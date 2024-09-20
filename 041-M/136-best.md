Big Admiral Dove

Medium

# A withdrawal operation may cause the position's Total Value Locked (TVL) to exceed the configured TVL limit.

## Summary

In the `SupplyLogic`, there exists a health factor validation step to prevent the position from being unhealthy after withdrawing. However, it doesn't check loan-to-value status so there is a risk that the position can be on the verge of being unhealthy.

## Vulnerability Detail

Generally, in lending pools, there are two factors to keep a position healthy - maximum LTV(Loan-To-Value) and Liquidation threshold.

The Pool protocol should prevent a position from being unhealthy instantly after withdrawing or borrowing, and that's why `ltv` is adopted in reserve configuration. (Check the [comments and code snippet](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/PoolLogic.sol#L155-L158) below)

```solidity
  function setReserveConfiguration(
    mapping(address => DataTypes.ReserveData) storage _reserves,
    address asset,
    address rateStrategyAddress,
    address source,
    DataTypes.ReserveConfigurationMap memory config
  ) public {
    ... ...
    // validation of the parameters: the LTV can
    // only be lower or equal than the liquidation threshold
    // (otherwise a loan against the asset would cause instantaneous liquidation)
    require(config.getLtv() <= config.getLiquidationThreshold(), PoolErrorsLib.INVALID_RESERVE_PARAMS);
    ... ...
  }
```

Therefore, `ltv` value of a reserve configuration represents the maximum loan ratio against the collateral within a position.

For borrowing operation, LTV check is performed by the following code snippet in the `ValidationLogic::validateBorrow()` function:

```solidity
  function validateBorrow(
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.ValidateBorrowParams memory params
  ) internal view {
    ... ...
    vars.collateralNeededInBaseCurrency = (vars.userDebtInBaseCurrency + vars.amountInBaseCurrency).percentDiv(vars.currentLtv);

    require(vars.collateralNeededInBaseCurrency <= vars.userCollateralInBaseCurrency, PoolErrorsLib.COLLATERAL_CANNOT_COVER_NEW_BORROW);
  }
```

Meanwhile, the withdrawing operation is designated to validate LTV via `ValidationLogic::validateHFAndLtv()` function as the post-withdrawal validation:

```solidity
  function validateHFAndLtv(
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.UserConfigurationMap memory userConfig,
    DataTypes.ExecuteWithdrawParams memory params
  ) internal view {
    DataTypes.ReserveData memory reserve = reservesData[params.asset];

    (, bool hasZeroLtvCollateral) = validateHealthFactor(_balances, reservesData, reservesList, userConfig, params.position, params.pool);

    require(!hasZeroLtvCollateral || reserve.configuration.getLtv() == 0, PoolErrorsLib.LTV_VALIDATION_FAILED);
  }

  function validateHealthFactor(
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.UserConfigurationMap memory userConfig,
    bytes32 position,
    address pool
  ) internal view returns (uint256, bool) {
    (,,,, uint256 healthFactor, bool hasZeroLtvCollateral) = GenericLogic.calculateUserAccountData(
      _balances,
      reservesData,
      reservesList,
      DataTypes.CalculateUserAccountDataParams({userConfig: userConfig, position: position, pool: pool})
    );

    require(healthFactor >= HEALTH_FACTOR_LIQUIDATION_THRESHOLD, PoolErrorsLib.HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD);

    return (healthFactor, hasZeroLtvCollateral);
  }
```

As can be seen from the `validateHFAndLtv()` function, actual LTV is not compared with the reserve configuration.

### Proof-Of-Concept

For test purposes, the `ValidationLogic::validateHealthFactor()` function is modified like the below:
```diff
  function validateHealthFactor(
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.UserConfigurationMap memory userConfig,
    bytes32 position,
    address pool
  ) internal view returns (uint256, bool) {
    (
+     uint256 totalCollateralInBaseCurrency, 
+     uint256 totalDebtInBaseCurrency,
+     uint256 avgLtv,,
      uint256 healthFactor,
      bool hasZeroLtvCollateral
    ) = GenericLogic.calculateUserAccountData(
      _balances,
      reservesData,
      reservesList,
      DataTypes.CalculateUserAccountDataParams({userConfig: userConfig, position: position, pool: pool})
    );

+   console.log("totalCollateralInBaseCurrency", totalCollateralInBaseCurrency);
+   console.log("totalDebtInBaseCurrency", totalDebtInBaseCurrency);
+   console.log("Position LTV", totalDebtInBaseCurrency.percentDiv(totalCollateralInBaseCurrency));
+   console.log("Configuration LTV", avgLtv);

    require(healthFactor >= HEALTH_FACTOR_LIQUIDATION_THRESHOLD, PoolErrorsLib.HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD);

    return (healthFactor, hasZeroLtvCollateral);
  }
```

And adds the new test case to `PoolBorrowTests.t.sol`:
```solidity
  function testWithdrawOverTVL() external {
    bytes32 pos = keccak256(abi.encodePacked(alice, 'index', uint256(0)));
    uint256 mintAmount = 1000 ether;
    uint256 borrowAmount = 100 ether;

    _mintAndApprove(alice, tokenA, mintAmount, address(pool));
    _mintAndApprove(bob, tokenB, mintAmount, address(pool));

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, mintAmount, 0);
    vm.stopPrank();

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, mintAmount, 0);

    DataTypes.ReserveData memory data = pool.getReserveData(address(tokenA));
    vm.expectEmit(true, true, true, false);
    emit PoolEventsLib.Borrow(address(tokenB), address(alice), pos, borrowAmount, data.borrowRate);
    pool.borrowSimple(address(tokenB), alice, borrowAmount, 0);

    assertEq(tokenB.balanceOf(address(pool)), mintAmount - borrowAmount);
    assertEq(pool.getDebt(address(tokenB), address(alice), 0), borrowAmount);
    assertEq(pool.totalDebt(address(tokenB)), borrowAmount);

    pool.withdrawSimple(address(tokenA), alice, 750 ether, 0);

    vm.stopPrank();
  }
```

After running the testcase, the following logs are displayed:
```bash
$ forge test --match-test testBorrowOverTVL -vvv
[⠒] Compiling...
[⠆] Compiling 11 files with Solc 0.8.19
[⠰] Solc 0.8.19 finished in 6.18s
Compiler run successful!

Ran 1 test for test/forge/core/pool/PoolBorrowTests.t.sol:PoolBorrowTests
[PASS] testBorrowOverTVL() (gas: 742984)
Logs:
  totalCollateralInBaseCurrency 25000000000
  totalDebtInBaseCurrency 20000000000
  Position LTV 8000
  Configuration LTV 7500

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.69ms (1.35ms CPU time)

Ran 1 test suite in 11.95ms (5.69ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

As can be seen in the logs, LTV of the position is greater than the reserve configuration and equals with the liquidation threshold, and the position will become unhealthy a seconds later.

## Impact

A withdrawing can result in the position being on the verge of unhealthy which will make it risky within a short period.

## Code Snippet

[pool/logic/ValidationLogic.sol#L239-L257](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ValidationLogic.sol#L239-L257)
[pool/logic/ValidationLogic.sol#L266-L278](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ValidationLogic.sol#L266-L278)

## Tool used

Manual Review

## Recommendation

I recommend updating `validateHFAndLtv()` and `validateHealthFactor()` functions like below:

```diff
  function validateHFAndLtv(
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.UserConfigurationMap memory userConfig,
    DataTypes.ExecuteWithdrawParams memory params
  ) internal view {
    DataTypes.ReserveData memory reserve = reservesData[params.asset];

-   (, bool hasZeroLtvCollateral) = validateHealthFactor(_balances, reservesData, reservesList, userConfig, params.position, params.pool);
+   (, bool hasZeroLtvCollateral, uint256 avgLtv, uint256 positionLtv) = validateHealthFactor(_balances, reservesData, reservesList, userConfig, params.position, params.pool);

-   require(!hasZeroLtvCollateral || reserve.configuration.getLtv() == 0, PoolErrorsLib.LTV_VALIDATION_FAILED);
+   require(!hasZeroLtvCollateral && positionLtv <= avgLtv, PoolErrorsLib.LTV_VALIDATION_FAILED);
  }

  function validateHealthFactor(
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.UserConfigurationMap memory userConfig,
    bytes32 position,
    address pool
- ) internal view returns (uint256, bool) {
+ ) internal view returns (uint256, bool, uint256, uint256) {
-   (,,,, uint256 healthFactor, bool hasZeroLtvCollateral) = GenericLogic.calculateUserAccountData(
+   (
+     uint256 totalCollateralInBaseCurrency, 
+     uint256 totalDebtInBaseCurrency,
+     uint256 avgLtv,,
+     uint256 healthFactor,
+     bool hasZeroLtvCollateral
    ) = GenericLogic.calculateUserAccountData(
      _balances,
      reservesData,
      reservesList,
      DataTypes.CalculateUserAccountDataParams({userConfig: userConfig, position: position, pool: pool})
    );

    require(healthFactor >= HEALTH_FACTOR_LIQUIDATION_THRESHOLD, PoolErrorsLib.HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD);

    uint256 positionLtv = totalDebtInBaseCurrency.percentDiv(totalCollateralInBaseCurrency);

-   return (healthFactor, hasZeroLtvCollateral);
+   return (healthFactor, hasZeroLtvCollateral, avgLtv, positionLtv);
  }
```
