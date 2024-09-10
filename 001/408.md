Curly Pineapple Armadillo

Medium

# Interest rates can be artificially lowered when liquidations occur

### Summary

When liquidations occur two calls to `updateInterestRates` are made. After the two interest rates are executed, debt tokens are transferred from the liquidator and collateral tokens are transferred to the liquidator. The issue is that a malicious liquidator can take advantage of that order of events by liquidating a position that has an asset that is used both for collateral and for debt. As a result, the malicious liquidator will artificially decrease the interest rate in an asset's reserve, harming all of the reserve's suppliers.

This will happen as if the debt token and the collateral tokens are the same, the second interest update, performed in `_burnCollateralTokens`, will override the first interest rate update. As a result, the liquidity added in the [first call to `updateInterestRates`](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L168) will not be considered when the interest rates are eventually calculated.

For example, 1 ETH of the collateral is liquidated from a position and the liquidator is going to have to pay 1 ETH for the debt. [`_repayDebtTokens`](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L161) is called, which reduces the total debt in `vars.debtReserveCache`, after that `updateInterestRates` is called. The interest rate is calculated by taking into account `the total asset balance of the pool + the liquidity added - the liquidity taken` (taken from [here](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/periphery/ir/DefaultReserveInterestRateStrategy.sol#L112)). The `_liquidityAdded` parameter of `updateInterestRates` is set to `vars.actualDebtToLiquidate`, because the debt tokens are yet to be transferred by the liquidator.
Now, `_burnCollateralTokens` is called, which also calls `updateInterestRates`. This time, `updateInterestRates` needs to update the interest rates based on the collateral transferred to the liquidator, thus `vars.actualCollateralToLiquidate` is set as the `_liquidityTaken` parameter of `updateInterestRates`.
However, if the debt and collateral tokens are the same, the second `updateInterestRates` will override the `_liquidityAdded` of the first `updateInterestRates`. This happens as the debt tokens are still not transferred to the pool and in the `the total asset balance of the pool + the liquidity added - the liquidity taken` calculation the asset balance of the pool will remain the same, but the liquidity added will be 0, instead of `vars.actualDebtToLiquidate`. As a result, interest rates will be lower than they should be, causing a loss of funds for all suppliers of a reserve.

### Root Cause

- In [`executeLiquidationCall`](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L94) the order of interest updates and token transfers allow for interest rates to be maliciously manipulated.

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

1. User liquidates 1 ETH of collateral, by paying 1 ETH of debt.
2. Overall the new interest rate updated, after the entire function has been implemented, should be calculated based on the `ETH.balanceOf(pool) + repaid_debt  - withdrawn_collateral => ETH.balanceOf(pool) + 1 ETH - 1 ETH`, however as the second call to update the interest rate will override the first it will be: `ETH.balanceOf(pool) + 0 - 1 ETH`
3. As a result, interest rates are lower than they should be and reserve suppliers accrue less interest than they should.

### Impact

Reserve suppliers accrue less interest than they are entitled to.

### PoC

_No response_

### Mitigation

Consider, transferring the debt tokens from the liquidator to the pool before calling `_burnCollateralTokens`.