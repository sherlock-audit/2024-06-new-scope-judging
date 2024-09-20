Big Admiral Dove

High

# Full Liquidation Won't Sweep the Whole Debts With Leaving Some, And Will Wrongly Set Borrowing as False

# Full Liquidation Won't Sweep the Whole Debts With Leaving Some, And Will Wrongly Set Borrowing as False

## Summary

When a liquidator tries to full liquidation (to cover full debts), there will leave some uncovered debts and the liquidation will wrongly set borrowing status of the debt asset as false.

## Vulnerability Detail

According to the [comment](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/interfaces/pool/IPoolSetters.sol#L143-L144) in `IPoolSetters.sol`, `debtToCover` parameter of the `liquidate()` function is intended to be debt assets, not shares.

> The caller (liquidator) covers `debtToCover` amount of debt of the user getting liquidated, and receives a proportionally amount of the `collateralAsset` plus a bonus to cover market risk

But the `_calculateDebt` function call in the `executeLiquidationCall()` do the operations on debt shares to calculate debt amount to cover and collateral amount to buy.

```solidity
  function _calculateDebt(
    DataTypes.ExecuteLiquidationCallParams memory params,
    uint256 healthFactor,
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage balances
  ) internal view returns (uint256, uint256) {
    uint256 userDebt = balances[params.debtAsset][params.position].debtShares;

    uint256 closeFactor = healthFactor > CLOSE_FACTOR_HF_THRESHOLD ? DEFAULT_LIQUIDATION_CLOSE_FACTOR : MAX_LIQUIDATION_CLOSE_FACTOR;

    uint256 maxLiquidatableDebt = userDebt.percentMul(closeFactor);

    uint256 actualDebtToLiquidate = params.debtToCover > maxLiquidatableDebt ? maxLiquidatableDebt : params.debtToCover;

    return (userDebt, actualDebtToLiquidate);
  }
```

According to this function, the return values `userDebt` and `actualDebtToLiquidate` are debt shares because they are not multiplied by borrow index.

Meanwhile, on the collateral reserve side, the `vars.userCollateralBalance` value that is provided as collateral balance to the `_calculateAvailableCollateralToLiquidate()` function, is collateral shares not assets. ([pool/logic/LiquidationLogic.sol#L136-L148](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L136-L148))

```solidity
    vars.userCollateralBalance = balances[params.collateralAsset][params.position].supplyShares;

    (vars.actualCollateralToLiquidate, vars.actualDebtToLiquidate, vars.liquidationProtocolFeeAmount) =
      _calculateAvailableCollateralToLiquidate(
        collateralReserve,
        vars.debtReserveCache,
        vars.actualDebtToLiquidate,
        vars.userCollateralBalance, // @audit-info Supply shares not assets
        vars.liquidationBonus,
        IPool(params.pool).getAssetPrice(params.collateralAsset),
        IPool(params.pool).getAssetPrice(params.debtAsset),
        IPool(params.pool).factory().liquidationProtocolFeePercentage()
      );
```

As there is no shares-to-assets conversion in the `_calculateAvailableCollateralToLiquidate()` function, the return values of the function `vars.actualCollateralToLiquidate`, `vars.actualDebtToLiquidate`, `vars.liquidationProtocolFeeAmount` are shares.

The remaining liquidation flow totally treat these share values as asset amounts. e.g. `_repayDebtTokens()` function calls the `repayDebt` function whose input is supposed to be assets:

```solidity
  function _repayDebtTokens( ... ) internal {
    uint256 burnt = balances[params.position].repayDebt(totalSupplies, vars.actualDebtToLiquidate, vars.debtReserveCache.nextBorrowIndex); // <-- @audit `vars.actualDebtToLiquidate` is shares at this moment
    vars.debtReserveCache.nextDebtShares = burnt;
  }
```

Thus, when a liquidator tries to cover full debts, the liquidation will leave `((borrowIndex - 1) / borrowIndex) * debtShares` debt shares and will set the borrowing status of the debt asset as false via the following [code snippet](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L150-L152).

```solidity
  function executeLiquidationCall(...) external {
    ... ...
    if (vars.userDebt == vars.actualDebtToLiquidate) {
      userConfig.setBorrowing(debtReserve.id, false);
    }
    ... ...
  }
```

### Proof-Of-Concept

To make a test case simple, I simplified the oracle price feeds like the below in the `CorePoolTests.sol` file:

```diff
  function _setUpCorePool() internal {
    ... ...
    oracleA = new MockV3Aggregator(8, 1e8);
-   oracleB = new MockV3Aggregator(18, 2 * 1e8);
+   oracleB = new MockV3Aggregator(8, 1e8);
    ... ...
  }
```

And created a new test file `PoolLiquidationTest2.sol`:

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import "../../../../lib/forge-std/src/console.sol";

import './PoolSetup.sol';

import {ReserveConfiguration} from './../../../../contracts/core/pool/configuration/ReserveConfiguration.sol';

import {UserConfiguration} from './../../../../contracts/core/pool/configuration/UserConfiguration.sol';

contract PoolLiquidationTest2 is PoolSetup {
  using UserConfiguration for DataTypes.UserConfigurationMap;
  using ReserveConfiguration for DataTypes.ReserveConfigurationMap;
  using ReserveConfiguration for DataTypes.ReserveData;

  address alice = address(1);
  address bob = address(2);

  uint256 mintAmountA = 200 ether;
  uint256 mintAmountB = 200 ether;
  uint256 supplyAmountA = 60 ether;
  uint256 supplyAmountB = 60 ether;
  uint256 borrowAmountB = 45 ether;

  function setUp() public {
    _setUpPool();
    pos = keccak256(abi.encodePacked(alice, 'index', uint256(0)));
  }

  // @audit-poc
  function testLiquidationInvalidUnits() external {
    oracleA.updateAnswer(1e8);
    oracleB.updateAnswer(1e8);

    _mintAndApprove(alice, tokenA, mintAmountA, address(pool));
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool));

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, supplyAmountA, 0);
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0);
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, borrowAmountB, 0);
    vm.stopPrank();

    // Advance time to make the position unhealthy
    vm.warp(block.timestamp + 360 days);
    oracleA.updateAnswer(1e8);
    oracleB.updateAnswer(1e8);

    // Print log of borrow rate before liquidation
    pool.forceUpdateReserve(address(tokenB));
    DataTypes.ReserveData memory reserveDataB = pool.getReserveData(address(tokenB));
    console.log("reserveDataB.borrowIndex before Liq.", reserveDataB.borrowIndex);

    DataTypes.PositionBalance memory positionBalance = pool.getBalanceRawByPositionId(address(tokenB), pos);
    console.log('debtShares Before Liq.', positionBalance.debtShares);

    DataTypes.UserConfigurationMap memory userConfig = pool.getUserConfiguration(alice, 0);
    console.log('TokenB isBorrowing Before Liq.', UserConfiguration.isBorrowing(userConfig, reserveDataB.id));

    vm.startPrank(bob);
    vm.expectEmit(true, true, true, false);
    emit PoolEventsLib.LiquidationCall(address(tokenA), address(tokenB), pos, 0, 0, bob);
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, 100 ether); // @audit Tries to cover full debts
    vm.stopPrank();

    positionBalance = pool.getBalanceRawByPositionId(address(tokenB), pos);
    console.log('debtShares After Liq.', positionBalance.debtShares);

    userConfig = pool.getUserConfiguration(alice, 0);
    console.log('TokenB isBorrowing After Liq.', UserConfiguration.isBorrowing(userConfig, reserveDataB.id));
  }
}
```
And here are the logs:
```bash
$ forge test --match-test testLiquidationInvalidUnits -vvv
[⠒] Compiling...
[⠊] Compiling 1 files with Solc 0.8.19
[⠒] Solc 0.8.19 finished in 4.83s
Compiler run successful!

Ran 1 test for test/forge/core/pool/PoolLiquidationPocTests2.t.sol:PoolLiquidationTest2
[PASS] testLiquidationInvalidUnits() (gas: 1172963)
Logs:
  reserveDataB.borrowIndex before Liq. 1252660064369089319656921640
  debtShares Before Liq. 45000000000000000000
  TokenB isBorrowing Before Liq. true
  debtShares After Liq. 9076447170314674990
  TokenB isBorrowing After Liq. false
```

As can be seen from the logs, there are significant amount of debts left but the borrowing flag was set as false.

## Impact

Wrongly setting borrowing status as false will affect the calculation of total debt amount, LTV and health factor, and this incorrect calculation will affect the whole ecosystem of a pool.

## Code Snippet

[pool/logic/LiquidationLogic.sol#L136](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L136)

[pool/logic/LiquidationLogic.sol#L264](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L264)

## Tool used

Manual Review

## Recommendation

Update the issued lines in the `LiquidationLogic.sol` file:

```diff
  function executeLiquidationCall(
    ... ...
  ) external {
    ... ...
-   vars.userCollateralBalance = balances[params.collateralAsset][params.position].supplyShares;
+   vars.userCollateralBalance = balances[params.collateralAsset][params.position].getSupplyBalance(collateralReserve.liquidityIndex);
    ... ...
  }

  function _calculateDebt(
    ... ...
  ) internal view returns (uint256, uint256) {
-   uint256 userDebt = balances[params.debtAsset][params.position].debtShares;
+   uint256 userDebt = balances[params.debtAsset][params.position].getDebtBalance(borrowIndex);
  }
```

I tried the above POC testcase to the update and the logs are:

```bash
$ forge test --match-test testLiquidationInvalidUnits -vv
[⠒] Compiling...
[⠊] Compiling 7 files with Solc 0.8.19
[⠒] Solc 0.8.19 finished in 5.90s
Compiler run successful!

Ran 1 test for test/forge/core/pool/PoolLiquidationPocTests2.t.sol:PoolLiquidationTest2
[PASS] testLiquidationInvalidUnits() (gas: 1137920)
Logs:
  reserveDataB.borrowIndex before Liq. 1252660064369089319656921640
  debtShares Before Liq. 45000000000000000000
  TokenB isBorrowing Before Liq. true
  debtShares After Liq. 0
  TokenB isBorrowing After Liq. false
```
