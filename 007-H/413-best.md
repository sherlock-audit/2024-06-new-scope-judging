Stale Tweed Seal

High

# Interest rate is updated before updating the debt when repaying debt

### Summary

Interest rate is updated before updating the debt when repaying debt in `BorrowLogic@executeRepay` leading to an incorrect total debt being used when calculating the new interest rates and causing suppliers to keep accruing interest based on the previous debt and even if there are no ongoing borrows anymore.

### Root Cause

[BorrowLogic](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L117-L161)
```solidity
  function executeRepay(
    DataTypes.ReserveData storage reserve,
    DataTypes.PositionBalance storage balances,
    DataTypes.ReserveSupplies storage totalSupplies,
    DataTypes.UserConfigurationMap storage userConfig,
    DataTypes.ExecuteRepayParams memory params
  ) external returns (DataTypes.SharesType memory payback) {
    DataTypes.ReserveCache memory cache = reserve.cache(totalSupplies);
    reserve.updateState(params.reserveFactor, cache);
    payback.assets = balances.getDebtBalance(cache.nextBorrowIndex);

    // Allows a user to max repay without leaving dust from interest.
    if (params.amount == type(uint256).max) {
      params.amount = payback.assets;
    }

    ValidationLogic.validateRepay(params.amount, payback.assets);

    // If paybackAmount is more than what the user wants to payback, the set it to the
    // user input (ie params.amount)
    if (params.amount < payback.assets) payback.assets = params.amount;

    reserve.updateInterestRates( // <==== Audit
      totalSupplies,
      cache,
      params.asset,
      IPool(params.pool).getReserveFactor(),
      payback.assets,
      0,
      params.position,
      params.data.interestRateData
    );

    // update balances and total supplies
    payback.shares = balances.repayDebt(totalSupplies, payback.assets, cache.nextBorrowIndex); // <==== Audit
    cache.nextDebtShares = totalSupplies.debtShares; // <==== Audit

    if (balances.getDebtBalance(cache.nextBorrowIndex) == 0) {
      userConfig.setBorrowing(reserve.id, false);
    }

    IERC20(params.asset).safeTransferFrom(msg.sender, address(this), payback.assets);
    emit PoolEventsLib.Repay(params.asset, params.position, msg.sender, payback.assets);
  }
```

[ReserveLogic](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L145-L182)
```solidity
  function updateInterestRates(
    DataTypes.ReserveData storage _reserve,
    DataTypes.ReserveSupplies storage totalSupplies,
    DataTypes.ReserveCache memory _cache,
    address _reserveAddress,
    uint256 _reserveFactor,
    uint256 _liquidityAdded,
    uint256 _liquidityTaken,
    bytes32 _position,
    bytes memory _data
  ) internal {
    UpdateInterestRatesLocalVars memory vars;

    vars.totalDebt = _cache.nextDebtShares.rayMul(_cache.nextBorrowIndex); // <==== Audit

    (vars.nextLiquidityRate, vars.nextBorrowRate) = IReserveInterestRateStrategy(_reserve.interestRateStrategyAddress)
      .calculateInterestRates(
      _position,
      _data,
      DataTypes.CalculateInterestRatesParams({
        liquidityAdded: _liquidityAdded, // <==== Audit
        liquidityTaken: _liquidityTaken,
        totalDebt: vars.totalDebt, // <==== Audit
        reserveFactor: _reserveFactor,
        reserve: _reserveAddress
      })
    );

    _reserve.liquidityRate = vars.nextLiquidityRate.toUint128();
    _reserve.borrowRate = vars.nextBorrowRate.toUint128();

    if (_liquidityAdded > 0) totalSupplies.underlyingBalance += _liquidityAdded.toUint128();
    else if (_liquidityTaken > 0) totalSupplies.underlyingBalance -= _liquidityTaken.toUint128();

    emit PoolEventsLib.ReserveDataUpdated(
      _reserveAddress, vars.nextLiquidityRate, vars.nextBorrowRate, _cache.nextLiquidityIndex, _cache.nextBorrowIndex
    );
  }
```

Interest rate is updated before repaying the debt and updating the cached `nextDebtShares` which is then used in the interest rate calculation causing it to return a wrong interest rate as it behaves like liquidity was just supplied by the borrower without a change in debt.

### Internal pre-conditions

N/A

### External pre-conditions

N/A

### Attack Path

1. Bob supplies `tokenB`
2. Alice supplies `tokenA`
3. Alice borrows `tokenB` causing the utilization goes up and interest rate is updated
4. Bob starts accruing interest
5. Alice fully repays `tokenB` but the interest rate is not updated correctly
6. Bob keeps accruing interest

### Impact

Bob keeps accruing interest rate based on the previous debt and even if there are no ongoing borrows and can withdraw it at the expense of other suppliers.

### PoC

```solidity
  function testRepay() external {
    _mintAndApprove(alice, tokenA, 3000 ether, address(pool));
    _mintAndApprove(alice, tokenB, 1000 ether, address(pool));
    _mintAndApprove(bob, tokenB, 5000 ether, address(pool));

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, 500 ether, 0);

    skip(12);
    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 1000 ether, 0);

    skip(12);
    oracleA.updateRoundTimestamp();
    oracleB.updateRoundTimestamp();
    pool.borrowSimple(address(tokenB), alice, 375 ether, 0);

    skip(12);
    pool.repaySimple(address(tokenB), type(uint256).max, 0);

    vm.stopPrank();

    bytes32 bobPos = keccak256(abi.encodePacked(bob, 'index', uint256(0)));
    uint256 bobSupplyAssetsBefore = pool.supplyAssets(address(tokenB), bobPos);

    skip(24 * 30 * 60 * 60);

    pool.forceUpdateReserve(address(tokenB));

    assertGt(pool.supplyAssets(address(tokenB), bobPos), bobSupplyAssetsBefore); // Bob accrued interest even if there are no borrows anymore

    vm.startPrank(bob);
    // Reverts because Bob shares with the accrued interest exceed the pool balance but
    // succeed if there were other suppliers.
    vm.expectRevert();
    pool.withdrawSimple(address(tokenB), bob, type(uint256).max, 0);

    vm.stopPrank();
  }
```

### Mitigation

```diff
diff --git a/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol b/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol
index 92806b1..c070fb1 100644
--- a/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol
+++ b/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol
@@ -136,6 +136,14 @@ library BorrowLogic {
     // user input (ie params.amount)
     if (params.amount < payback.assets) payback.assets = params.amount;

+    // update balances and total supplies
+    payback.shares = balances.repayDebt(totalSupplies, payback.assets, cache.nextBorrowIndex);
+    cache.nextDebtShares = totalSupplies.debtShares;
+
+    if (balances.getDebtBalance(cache.nextBorrowIndex) == 0) {
+      userConfig.setBorrowing(reserve.id, false);
+    }
+
     reserve.updateInterestRates(
       totalSupplies,
       cache,
@@ -147,14 +155,6 @@ library BorrowLogic {
       params.data.interestRateData
     );

-    // update balances and total supplies
-    payback.shares = balances.repayDebt(totalSupplies, payback.assets, cache.nextBorrowIndex);
-    cache.nextDebtShares = totalSupplies.debtShares;
-
-    if (balances.getDebtBalance(cache.nextBorrowIndex) == 0) {
-      userConfig.setBorrowing(reserve.id, false);
-    }
-
     IERC20(params.asset).safeTransferFrom(msg.sender, address(this), payback.assets);
     emit PoolEventsLib.Repay(params.asset, params.position, msg.sender, payback.assets);
   }
```