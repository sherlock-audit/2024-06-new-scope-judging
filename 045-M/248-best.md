Careful Fleece Pike

Medium

# A liquidator could left unbacked debt behind

### Summary

Improper implementation for the liquidation in case of `vars.maxCollateralToLiquidate > userCollateralBalance` will cause leaving unbacked debt behind for the pool.

### Root Cause

During the liquidation process, the amount of collateral (`collateralAmount`) to take from a position and the amount of debt (`debtAmountNeeded`) to erase from a position are calculated in `_calculateAvailableCollateralToLiquidate`

https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L328

```solidity
  function _calculateAvailableCollateralToLiquidate(
    DataTypes.ReserveData storage collateralReserve,
    DataTypes.ReserveCache memory debtReserveCache,
    uint256 debtToCover,
    uint256 userCollateralBalance,
    uint256 liquidationBonus,
    uint256 collateralPrice,
    uint256 debtAssetPrice,
    uint256 liquidationProtocolFeePercentage
  ) internal view returns (uint256, uint256, uint256) {
    AvailableCollateralToLiquidateLocalVars memory vars;

    vars.collateralPrice = collateralPrice; // oracle.getAssetPrice(collateralAsset);
    vars.debtAssetPrice = debtAssetPrice; // oracle.getAssetPrice(debtAsset);

    vars.collateralDecimals = collateralReserve.configuration.getDecimals();
    vars.debtAssetDecimals = debtReserveCache.reserveConfiguration.getDecimals();

    unchecked {
      vars.collateralAssetUnit = 10 ** vars.collateralDecimals;
      vars.debtAssetUnit = 10 ** vars.debtAssetDecimals;
    }

    // This is the base collateral to liquidate based on the given debt to cover
    vars.baseCollateral = ((vars.debtAssetPrice * debtToCover * vars.collateralAssetUnit)) / (vars.collateralPrice * vars.debtAssetUnit);

    vars.maxCollateralToLiquidate = vars.baseCollateral.percentMul(liquidationBonus);

    if (vars.maxCollateralToLiquidate > userCollateralBalance) {
1>    vars.collateralAmount = userCollateralBalance;
1>    vars.debtAmountNeeded = (
1>      (vars.collateralPrice * vars.collateralAmount * vars.debtAssetUnit) / (vars.debtAssetPrice * vars.collateralAssetUnit)
1>    ).percentDiv(liquidationBonus);
    } else {
2>    vars.collateralAmount = vars.maxCollateralToLiquidate;
2>    vars.debtAmountNeeded = debtToCover;
    }

    if (liquidationProtocolFeePercentage != 0) {
      vars.bonusCollateral = vars.collateralAmount - vars.collateralAmount.percentDiv(liquidationBonus);
      vars.liquidationProtocolFee = vars.bonusCollateral.percentMul(liquidationProtocolFeePercentage);
      return (vars.collateralAmount - vars.liquidationProtocolFee, vars.debtAmountNeeded, vars.liquidationProtocolFee);
    } else {
      return (vars.collateralAmount, vars.debtAmountNeeded, 0);
    }
  }
```

There are two cases in the liquidation process:
1. The liquidator clears all the debt of the position. This is the case of `vars.maxCollateralToLiquidate <= userCollateralBalance`, and the code at `2>` will run.
2. The liquidator takes all the collateral of the position, but it does not guarantee all the debt of the position will be cleared. This is the case of  `vars.maxCollateralToLiquidate > userCollateralBalance`. and the code at `1>` will run.

In the second case, if the collateral is the last active collateral of the position, then the debt will be unbacked.

### Internal pre-conditions

1. The position has only one active collateral
2. The position is liquidatable
3. `vars.maxCollateralToLiquidate > userCollateralBalance`

### External pre-conditions

_No response_

### Attack Path

Liquidate a position with the internal pre-conditions that are met.

### Impact

- The position has unbacked debt
- The unbacked can not be liquidated, because the amount of collateral in the position is zero.
- The unbacked debt still accrues interest.
- Having unbacked debt will affect the `borrowUsageRatio` and `supplyUsageRatio`, which in turn affects the `liquidityRate` and `borrowRate` negatively.

### PoC

Run command: `forge test --match-path test/PoC/PoC.t.sol -vv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import {PoolSetup} from 'test/forge/core/pool/PoolSetup.sol';
import {console} from 'lib/forge-std/src/Test.sol';

contract PoC is PoolSetup {
  address alice = makeAddr('alice');
  address bob = makeAddr('bob');

  function setUp() public {
    _setUpPool();
  }

  function testPoC() external {
    uint256 mintAmount = 3 ether;
    uint256 borrowAmount = 1 ether;

    _mintAndApprove(alice, tokenA, mintAmount, address(pool));
    _mintAndApprove(alice, tokenB, mintAmount, address(pool));

    _mintAndApprove(bob, tokenB, mintAmount, address(pool));

    vm.prank(bob);
    pool.supplySimple(address(tokenB), bob, borrowAmount, 0);

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, mintAmount, 0);
    pool.borrowSimple(address(tokenB), alice, borrowAmount, 0);
    vm.stopPrank();

    // Simulate the condition `vars.maxCollateralToLiquidate > userCollateralBalance`
    // Price of borrow token increases from 2e8 to 4e8
    oracleB.updateAnswer(4e8);

    bytes32 alicePos = keccak256(abi.encodePacked(alice, 'index', uint256(0)));
    vm.prank(bob);
    pool.liquidateSimple(address(tokenA), address(tokenB), alicePos, type(uint256).max);

    console.log("Collateral shares: %e", pool.getBalanceRawByPositionId(address(tokenA), alicePos).supplyShares);
    console.log("Debt shares: %e", pool.getBalanceRawByPositionId(address(tokenB), alicePos).debtShares);
  }
}
```
Logs:

```bash
  Collateral shares: 0e0
  Debt shares: 2.85714285714285714e17
```

### Mitigation

In case of `vars.maxCollateralToLiquidate > userCollateralBalance`, the liquidator also has to pay for all the `debtToCover`

```diff
    if (vars.maxCollateralToLiquidate > userCollateralBalance) {
      vars.collateralAmount = userCollateralBalance;
-     vars.debtAmountNeeded = (
-       (vars.collateralPrice * vars.collateralAmount * vars.debtAssetUnit) / (vars.debtAssetPrice * vars.collateralAssetUnit)
-     ).percentDiv(liquidationBonus);
    } else {
      vars.collateralAmount = vars.maxCollateralToLiquidate;
-     vars.debtAmountNeeded = debtToCover;
    }
+   vars.debtAmountNeeded = debtToCover;
```

If the above mitigation is implemented, add a `minCollateralTaken` parameter in the liquidation function to avoid race condition during the liquidation.