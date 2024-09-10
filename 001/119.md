Acrobatic Rainbow Grasshopper

Medium

# Users can't withdraw all of their funds upon withdrawing ETH using the NFT position manager

## Summary
Users can't withdraw all of their funds upon withdrawing ETH using the NFT position manager
## Vulnerability Detail
Users can withdraw ETH directly using `NFTPositionManager::withdrawETH()`. Upon withdrawing, we have the following line in the Pool contract:
```solidity
if (params.amount == type(uint256).max) params.amount = balance;
```
The goal of this line is for users to be able to provide all of their balance as it is pretty much impossible to do so otherwise due to the interest accrued based on `block.timestamp`. When using the NFT position manager, upon withdrawing the WETH, we have this line:
```solidity
weth.withdraw(params.amount);
```
The provided amount value was not changed in any way, shape or form, thus it would be `type(uint256).max` in the case explained above. This would obviously revert as neither the `WETH` contract has that amount of funds, nor the NFT position manager contract has access to such an amount of funds. As the funds grow with `block.timestamp` which is an impossible value to predict, upon the user trying to predict that value, he would either choose a lower value causing him to not withdraw all his funds or he would provide a bigger value causing a revert.
## Impact
Users can't withdraw all of their funds upon withdrawing ETH using the NFT position manager
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTPositionManager.sol#L105-L113
## Tool used

Manual Review

## Recommendation
Handle the case where the provided amount input is `type(uint256).max`