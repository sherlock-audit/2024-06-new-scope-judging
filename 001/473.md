Ancient Gingerbread Rooster

High

# Wrong calculation of supply/debt balance of a position, disrupting core system functionalities

## Summary
There is an error in the calculation of the supply/debt balance of a position, impacting a wide range of operations across the system, including core lending features.

## Vulnerability Detail
`PositionBalanceConfiguration` library have 2 methods to provide supply/debt balance of a position as follows:

```solidity
  function getSupplyBalance(DataTypes.PositionBalance storage self, uint256 index) public view returns (uint256 supply) {
    uint256 increase = self.supplyShares.rayMul(index) - self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex);
>   return self.supplyShares + increase;
  }

  function getDebtBalance(DataTypes.PositionBalance storage self, uint256 index) internal view returns (uint256 debt) {
    uint256 increase = self.debtShares.rayMul(index) - self.debtShares.rayMul(self.lastDebtLiquidtyIndex);
>   return self.debtShares + increase;
  }
```

The implementation contains a critical error as it returns the `share amount` rather than the `asset amount` held. This issue is evident when the function utilizes `index`, same as `self.lastSupplyLiquidityIndex` or `self.lastDebtLiquidityIndex`. Each function returns `self.supplyShares` and `self.debtShares`, which are share amounts, while the caller expects accurate asset balances. A similar issue occurs when a different index is used, still resulting in an incorrect balance that is significantly lower than the actual balance.

Below I provide a sample scenario to check supply balance (`getSupplyBalance` using same liquidity index):
1. Suppose `position.lastSupplyLiquidtyIndex` = `2 RAY` (2e27)  (Time passed as the liquidity increased).
2. Now position is supplied `2 RAY` of assets, it got `1 RAY` (2RAY.rayDiv(2RAY)) shares minted.
3. Then `position.getSupplyBalance(2 RAY)` returns `1 RAY` while we expect `2 RAY` which is correct balance.

Below is a foundary PoC to validate one live example: failure to fully repay with type(uint256).max due to balance error. Full script can be found [here](https://gist.github.com/worca333/8103ca8527e918b4fc8ab06b71ac798a).
```solidity
  function testRepayFailWithUint256MAX() external {
    _mintAndApprove(alice, tokenA, 4000 ether, address(pool));

    // Set the reserve factor to 1000 bp (10%)
    poolFactory.setReserveFactor(10_000);

    // Alice supplies and borrows tokenA from the pool
    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 2000 ether, 0);
    pool.borrowSimple(address(tokenA), alice, 800 ether, 0);

    vm.warp(block.timestamp + 10 minutes);

    assertGt(pool.getDebt(address(tokenA), alice, 0), 0);
    vm.stopPrank();

    // borrow again: this will update reserve.lastDebtLiquidtyIndex
    vm.startPrank(alice);
    pool.borrowSimple(address(tokenA), alice, 20 ether, 0);
    vm.stopPrank();

    pool.forceUpdateReserve(address(tokenA));

    console.log("Debt before repay: ", pool.getDebt(address(tokenA), alice, 0));
    vm.startPrank(alice);
    tokenA.approve(address(pool), UINT256_MAX);
    pool.repaySimple(address(tokenA), UINT256_MAX, 0);
    console.log("Debt after  repay: ", pool.getDebt(address(tokenA), alice, 0));

    console.log("Assert: Debt still exists after full-repay with UINT256_MAX");
    assertNotEq(pool.getDebt(address(tokenA), alice, 0), 0);
    vm.stopPrank();
  }
```

Run the test by 
```bash
forge test --mt testRepayFailWithUint256MAX -vvv
```

Logs:
```bash
[PASS] testRepayFailWithUint256MAX() (gas: 567091)
Logs:
  Debt before repay:  819999977330884982376
  Debt after  repay:  929433690028148
  Assert: Debt still exists after full-repay with UINT256_MAX

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.68ms (1.18ms CPU time)

Ran 1 test suite in 281.17ms (4.68ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact
Since these functions are integral to the core pool/position logic and are utilized extensively across the system, the impacts are substantial.
1. In `SupplyLogic.executeWithdraw`, withdrawl is processed based on wrong position supply balance which potentially could fail.
```solidity
  function executeWithdraw(
    ...
  ) external returns (DataTypes.SharesType memory burnt) {
    DataTypes.ReserveData storage reserve = reservesData[params.asset];
    DataTypes.ReserveCache memory cache = reserve.cache(totalSupplies);
    reserve.updateState(params.reserveFactor, cache);

>   uint256 balance = balances[params.asset][params.position].getSupplyBalance(cache.nextLiquidityIndex);
    ...
```

2. In `BorrowLogic.executeRepay`, it would fail
  - to do full-repay because `payback.assets` is not the total debt amount
  - to do `setBorrwing(reserve.id, false)` because `getDebtBalance` almost unlikely goes 0, as it can't do full repay
```solidity
  function executeRepay(
    ...
  ) external returns (DataTypes.SharesType memory payback) {
    DataTypes.ReserveCache memory cache = reserve.cache(totalSupplies);
    reserve.updateState(params.reserveFactor, cache);
>   payback.assets = balances.getDebtBalance(cache.nextBorrowIndex);

    // Allows a user to max repay without leaving dust from interest.
    if (params.amount == type(uint256).max) {
>     params.amount = payback.assets;
    }

    ...

>   if (balances.getDebtBalance(cache.nextBorrowIndex) == 0) {
      userConfig.setBorrowing(reserve.id, false);
    }

    IERC20(params.asset).safeTransferFrom(msg.sender, address(this), payback.assets);
    emit PoolEventsLib.Repay(params.asset, params.position, msg.sender, payback.assets);
  }
```

3. In `NFTPositionManagerSetter._supply` and `NFTPositionManagerSetter._borrow`, they call `NFTRewardsDistributor._handleSupplies` and `NFTRewardsDistributor._handleDebt` with wrong balance amounts which would lead to incorrect reward distribution.
```solidity
  function _supply(AssetOperationParams memory params) internal nonReentrant {
    if (params.amount == 0) revert NFTErrorsLib.ZeroValueNotAllowed();
    if (params.tokenId == 0) {
      if (msg.sender != _ownerOf(_nextId - 1)) revert NFTErrorsLib.NotTokenIdOwner();
      params.tokenId = _nextId - 1;
    }

    IPool pool = IPool(_positions[params.tokenId].pool);

    IERC20(params.asset).forceApprove(address(pool), params.amount);
    pool.supply(params.asset, address(this), params.amount, params.tokenId, params.data);

    // update incentives
>   uint256 balance = pool.getBalance(params.asset, address(this), params.tokenId);
    _handleSupplies(address(pool), params.asset, params.tokenId, balance);

    emit NFTEventsLib.Supply(params.asset, params.tokenId, params.amount);
  }

  function _borrow(AssetOperationParams memory params) internal nonReentrant {
    if (params.target == address(0)) revert NFTErrorsLib.ZeroAddressNotAllowed();
    if (params.amount == 0) revert NFTErrorsLib.ZeroValueNotAllowed();
    if (params.tokenId == 0) {
      if (msg.sender != _ownerOf(_nextId - 1)) revert NFTErrorsLib.NotTokenIdOwner();
      params.tokenId = _nextId - 1;
    }

    // check permissions
    _isAuthorizedForToken(params.tokenId);

    IPool pool = IPool(_positions[params.tokenId].pool);
    pool.borrow(params.asset, params.target, params.amount, params.tokenId, params.data);

    // update incentives
>   uint256 balance = pool.getDebt(params.asset, address(this), params.tokenId);
    _handleDebt(address(pool), params.asset, params.tokenId, balance);

    emit NFTEventsLib.Borrow(params.asset, params.amount, params.tokenId);
  }
```

4. In `NFTPositionManagerSetter._repay`, wrong balance is used to estimate debt status and refunds.
  - It will almost likely revert with `NFTErrorsLib.BalanceMisMatch` because `debtBalance` is share amount versus `repaid.assets` is asset amount
  - `currentDebtBalance` will never go 0 because it almost unlikely gets repaid in full, hence refund never happens
  - `_handleDebt` would work wrongly due to incorrect balance 
```solidity
  function _repay(AssetOperationParams memory params) internal nonReentrant {
    ...
>   uint256 previousDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);
    DataTypes.SharesType memory repaid = pool.repay(params.asset, params.amount, params.tokenId, params.data);
>   uint256 currentDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);

    if (previousDebtBalance - currentDebtBalance != repaid.assets) {
      revert NFTErrorsLib.BalanceMisMatch();
    }

    if (currentDebtBalance == 0 && repaid.assets < params.amount) {
      asset.safeTransfer(msg.sender, params.amount - repaid.assets);
    }

    // update incentives
    _handleDebt(address(pool), params.asset, params.tokenId, currentDebtBalance);

    emit NFTEventsLib.Repay(params.asset, params.amount, params.tokenId);
  }
```

5. In `CuratedVault.totalAssets`, it returns wrong asset amount.
```solidity
  function totalAssets() public view override returns (uint256 assets) {
    for (uint256 i; i < withdrawQueue.length; ++i) {
>     assets += withdrawQueue[i].getBalanceByPosition(asset(), positionId);
    }
  }
```

## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L126-L140

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L118

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L126

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L44-L82

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L119-L121

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVault.sol#L368-L372

## Tool used
Manual Review, Foundary

## Recommendation
The `getSupplyBalance` and `getDebtBalance` functions need an update to accurately reflect the balance. Referring to `getSupplyBalance` and `getDebtBalance` functions from `ReserveSuppliesConfiguration`, we can make updates as following:

```diff
  function getSupplyBalance(DataTypes.PositionBalance storage self, uint256 index) public view returns (uint256 supply) {
-   uint256 increase = self.supplyShares.rayMul(index) - self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex);
-   return self.supplyShares + increase;
+   return self.supplyShares.rayMul(index);
  }

  function getDebtBalance(DataTypes.PositionBalance storage self, uint256 index) internal view returns (uint256 debt) {
-   uint256 increase = self.debtShares.rayMul(index) - self.debtShares.rayMul(self.lastDebtLiquidtyIndex);
-   return self.debtShares + increase;
+   return self.debtShares.rayMul(index);
  }
```