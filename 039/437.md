Clever Ebony Halibut

Medium

# Undistributed Rewards Lost in NFTRewardsDistributor Due to Lack of Rollover Mechanism

## Summary

The `notifyRewardAmount` function in `NFTRewardsDistributor.sol` fails to roll over undistributed rewards from previous epochs, leading to a loss of rewards when no users are supplying to the pool during an entire epoch.

## Vulnerability Detail

The `notifyRewardAmount` function in `NFTRewardsDistributor.sol` is responsible for setting up new reward periods and determining the reward rate. However, it does not account for undistributed rewards from previous periods when starting a new one in case totalSupply was 0.

```js
function notifyRewardAmount(uint256 reward, address pool, address asset, bool isDebt) external onlyRole(REWARDS_ALLOCATOR_ROLE) {
rewardsToken.transferFrom(msg.sender, address(this), reward);

    bytes32 _assetHash = assetHash(pool, asset, isDebt);
    _updateReward(0, _assetHash);

    if (block.timestamp >= periodFinish[_assetHash]) {

@>> rewardRate[_assetHash] = reward.div(rewardsDuration);
} else {
uint256 remaining = periodFinish[_assetHash].sub(block.timestamp);
uint256 leftover = remaining.mul(rewardRate[_assetHash]);
@>> rewardRate[_assetHash] = reward.add(leftover).div(rewardsDuration);
}
// ... (remaining code)
}
```

The issue arises in two scenarios:

1. When a new period starts and prev finished (`block.timestamp >= periodFinish[_assetHash]`):

   - The function sets `rewardRate[_assetHash] = reward.div(rewardsDuration)`
   - This calculation only considers the new reward amount, ignoring any undistributed rewards from the previous period in case totalSupply was 0.

2. When updating an ongoing period:
   - The function calculates `leftover = remaining.mul(rewardRate[_assetHash])`
   - While this accounts for remaining rewards in the current period, it doesn't include undistributed rewards for the elepsed time.

In both cases, if no users weren't supplying/borrowing , the rewards allocated for that period are effectively lost for the period that were totalSupply was 0. They remain in the contract but are not factored into future reward calculations or distributions.

This issue is particularly problematic for newly launched or less popular pools, where there might be periods of low or no activity. It can lead to a significant discrepancy between the total rewards allocated to a pool and the rewards actually distributed to users over time.

## Impact

The vulnerability results in the loss of undistributed rewards, reducing the effective reward rate for users and potentially diminishing the incentive mechanism's effectiveness.

## Code Snippet
- https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L125
## Tool used

Manual Review

## Recommendation

Implement a rollover mechanism to:

1. Update reward rate when totalSupply changes from zero to non-zero
2. Preserve undistributed rewards when totalSupply becomes zero
   This ensures no rewards are lost during periods of inactivity.
