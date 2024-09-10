Scruffy Dijon Chinchilla

Medium

# Users might not receive reward token as `getReward()` doesnt revert for failure transactions

### Summary

Users can call `getReward()` without actually receiving tokens if a no-revert-on-failure token is used and there is no check on the return value of this transfer, causing a portion of their claimable tokens to become unclaimable. One of the confirmed reward tokens is `ZERO` which has a boolean return value because it adopts the `IERC20` interface for the `transfer()` function. If the transaction fails then it will happen silently, this causes the portion of the reward for the user to still be reduced but the user does not receive the reward.

Note
Unchecked return value of `transfer()` function is also found in :

1. In [NFTPositionManager.sol:134](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManager.sol#L134)
2. In [NFTRewardsDistributor.sol:126](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L126)

### Root Cause

In [NFTRewardsDistributor.sol:88-96](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L88-L96) there is missing check on return value for `transfer()`

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

Users will loss reward token and the portion of claimable reward

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

Consider add check for return value 

```solidity
bool success = rewardsToken.transfer(ownerOf(tokenId), reward);
require(success, 'transfer failed');
```