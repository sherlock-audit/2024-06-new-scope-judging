Acrobatic Rainbow Grasshopper

Medium

# Full liquidations are possible at an earlier point due to off by one error

## Summary
Full liquidations are possible at an earlier point due to off by one error
## Vulnerability Detail
Upon liquidations, we call `LiquidationLogic::_calculateDebt()`, there we have this line:
```solidity
    uint256 closeFactor = healthFactor > CLOSE_FACTOR_HF_THRESHOLD ? DEFAULT_LIQUIDATION_CLOSE_FACTOR : MAX_LIQUIDATION_CLOSE_FACTOR;
```
If the health factor is above 0.95e18, we can liquidate half of the user's debt (5000 BIPS). If equal or below, we can liquidate the whole user's debt (10_000 BIPS). However, that is incorrect. We should be able to liquidate the whole debt only if the helath factor is below 0.95e18, not if they are equal. This is also seen in the natspec of the function:

>@dev If the Health Factor is below CLOSE_FACTOR_HF_THRESHOLD, the close factor is increased to MAX_LIQUIDATION_CLOSE_FACTOR

This is an extremely important function in the code so such an error could be very detrimental.
## Impact
Full liquidations are possible at an earlier point due to off by one error
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L259-L273
## Tool used

Manual Review

## Recommendation
```diff
-uint256 closeFactor = healthFactor > CLOSE_FACTOR_HF_THRESHOLD ? DEFAULT_LIQUIDATION_CLOSE_FACTOR : MAX_LIQUIDATION_CLOSE_FACTOR;
+uint256 closeFactor = healthFactor >= CLOSE_FACTOR_HF_THRESHOLD ? DEFAULT_LIQUIDATION_CLOSE_FACTOR : MAX_LIQUIDATION_CLOSE_FACTOR;
```