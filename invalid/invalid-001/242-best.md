Fit Bone Eel

High

# The user will receive no rewards

### Summary

In the `_updateReward()` function, the protocol first updates `lastUpdateTime[_assetHash]` and then calls `earned()` to calculate the user's rewards. 
```solidity
  function _updateReward(uint256 _tokenId, bytes32 _assetHash) internal {
    rewardPerTokenStored[_assetHash] = rewardPerToken(_assetHash);
    lastUpdateTime[_assetHash] = lastTimeRewardApplicable(_assetHash);
    if (_tokenId != 0) {
      rewards[_tokenId][_assetHash] = earned(_tokenId, _assetHash);
      userRewardPerTokenPaid[_tokenId][_assetHash] = rewardPerTokenStored[_assetHash];
    }
  }

```


The protocol calculates `rewardPerToken` as follows:

```solidity
rewardPerTokenStored[_assetHash].add(
  lastTimeRewardApplicable(_assetHash).sub(lastUpdateTime[_assetHash]).mul(rewardRate[_assetHash]).mul(1e18).div(
    _totalSupply[_assetHash]
  )
)
```

Since `lastUpdateTime[_assetHash]` has already been updated earlier, the value of `lastTimeRewardApplicable(_assetHash).sub(lastUpdateTime[_assetHash])` will be 0, resulting in `rewardPerToken` being 0. This leads to the user receiving no rewards.

### Root Cause
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L153
`lastUpdateTime[_assetHash]` is updated before calculating the rewards, which results in no accumulated time. Consequently, the calculated reward is 0.

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

The user called getReward(), but there was no reward.


### Impact

The user receives no rewards.

### PoC

```solidity
  function getReward(uint256 tokenId, bytes32 _assetHash) public nonReentrant returns (uint256 reward) {
    _updateReward(tokenId, _assetHash);
    reward = rewards[tokenId][_assetHash];
    if (reward > 0) {
      rewards[tokenId][_assetHash] = 0;
      rewardsToken.transfer(ownerOf(tokenId), reward);
      emit RewardPaid(tokenId, ownerOf(tokenId), reward);
    }
  }
```

### Mitigation

After calculating the reward, update the lastUpdateTime.
