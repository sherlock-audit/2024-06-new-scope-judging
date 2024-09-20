Curly Pineapple Armadillo

Medium

# `NFTRewardsDistributor` assumes that all tokens supported by Pools will have 18 decimals, causing wrong reward calculations

### Summary

The `rewardPerToken` function of `NFTRewardsDistributor` makes the wrong assumprion that all tokens supported by pools will have 18 decimals when calculating reward amounts. This is an issue as USDC which has 6 decimals of precision (and other tokens with smaller decimals) will be supported by the protocol, thus rewards for such tokens will be calculated wrongly.

### Root Cause

- In [`rewardPerToken`](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L82) 1e18 is used to account for the scaling of supplied tokens(not the reward token) when caclulating the reward per token.

### Internal pre-conditions

1. The token that the rewards distributor calculates rewards for must have decimals smaller than 18, like USDC.

### External pre-conditions

_No response_

### Attack Path

1. `NFTRewardsDistributor` calculates rewards per token for a USDC reserve.
2. `rewardPerToken` is called and reward per token is calculated as: `time_applicable * reward_rate * 1e18 / total_suply_of_usdc`, thus the calculation in decimals will be: `time_applicable * decimals_of_reward * 1e18 / decimals_of_usdc`, which is essentially `1e18 * 1e18 / 1e6` = a result with 1e30 decimals of precision when the correct result should have the decimals of the reward token: 1e18. 

### Impact

Even though the `earned` function balances out the high number of decimals by also scaling down 18 decimals instead of the supply tokens decimals, `rewardPerToken` will still return a much higher number than intended, which may cause users to assuming that the reward per token is much higher than it actually is.

### PoC

_No response_

### Mitigation

In both `earned` and `rewardPerToken` use the decimals of the supplied asset for scaling.