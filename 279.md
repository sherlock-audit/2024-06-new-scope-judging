Joyous Cedar Tortoise

Medium

# When the collateral token is BNB, full liquidations will revert due to sending 0 amount

### Summary

BNB has a check in it's `transfer()` function to revert for 0 amounts

The issue is that during complete liquidations (taking all the collateral), the following [[code](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L178-L189)](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L178-L189) will send a 0 amount, which will revert the whole liquidation call

```solidity
// Transfer fee to treasury if it is non-zero
    if (vars.liquidationProtocolFeeAmount != 0) {
      uint256 liquidityIndex = collateralReserve.getNormalizedIncome();
      uint256 scaledDownLiquidationProtocolFee = vars.liquidationProtocolFeeAmount.rayDiv(liquidityIndex);
      uint256 scaledDownUserBalance = balances[params.collateralAsset][params.position].supplyShares;

      if (scaledDownLiquidationProtocolFee > scaledDownUserBalance) {
        vars.liquidationProtocolFeeAmount = scaledDownUserBalance.rayMul(liquidityIndex);
      }

      IERC20(params.collateralAsset).safeTransfer(IPool(params.pool).factory().treasury(), vars.liquidationProtocolFeeAmount);
    }

```

Even if the `vars.liquidationProtocolFeeAmount != 0`

since `scaledDownUserBalance == 0` (user has 0 collateral since the liquidator took all of it)

It will update `vars.liquidationProtocolFeeAmount == scaledDownUserBalance.rayMul(liquidityIndex)`, which is 0

Therefore trying to transfer the 0 amount will revert

### Root Cause

During a full liquidation, it will attempt to transfer a 0 amount [here](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L178-L189), which will revert the whole liquidation call

### Internal pre-conditions

Collateral token of user is BNB

### External pre-conditions

Liquidator attempts to take all the collateral

### Attack Path

_No response_

### Impact

Liquidation reverts, increasing the risk of accumulating bad debt

### PoC

_No response_

### Mitigation

Implement the 0 amount check right before the transfer to prevent this issue