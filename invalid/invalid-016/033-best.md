Loud Rosewood Oyster

Medium

# USDC Fee-on-Transfer Will Result in Incorrect Share Minting in `SupplyLogic`

## Summary
the current supply logic will mint more shares than should have to the depositing user, if the asset is a fee on transfer token.
## Vulnerability Detail
if a pool uses USDC as a reserve asset, if USDC starts charging fees on transfer, the current supply logic will deposit and mint the wrong amount of shares to the user.
[SupplyLogic::executeSupply](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L58-L94):
```solidity
    IERC20(params.asset).safeTransferFrom(msg.sender, address(this), params.amount);
    bool isFirst;
    (isFirst, minted.shares) = balance.depositCollateral(totalSupplies, params.amount, cache.nextLiquidityIndex);
```
Note that USDC has been listed in the readme as possible tokens to integrate with the contracts.
[_HERE_](https://github.com/sherlock-audit/2024-06-new-scope?tab=readme-ov-file#q-if-you-are-integrating-tokens-are-you-allowing-only-whitelisted-tokens-to-work-with-the-codebase-or-any-complying-with-the-standard-are-they-assumed-to-have-certain-properties-eg-be-non-reentrant-are-there-any-types-of-weird-tokens-you-want-to-integrate:~:text=Only%20standard%20ERC20%20tokens%20%2B%20USDC%20and%20BNB%20are%20in%2Dscope)
## Impact
Will mint more shares to the user
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L81-L83
## Tool used

Manual Review

## Recommendation
should store the initial contract asset balance before getting the assets from the caller, then deduct the initial balance from the new balance after the transfer. param.amount should be updated to the difference.