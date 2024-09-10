Careful Fleece Pike

Medium

# An attacker will prevent rewards from accruing when a low decimals token (`USDC`) being used as `rewardsToken` in `NFTPositionManager`

### Summary

When a low decimals token (`USDC`) being used as `rewardsToken` in `NFTPositionManager`, rounding to zero in `NFTRewardsDistributor#rewardPerToken` will cause the rewards do not accrue as an attacker will update the rewards every block.

### Root Cause

From the contest's `README`, the protocol is planning to support `USDC`, which is a low decimals token. When a low decimals token being used as `rewardsToken` in `NFTPositionManager`, under reasonable conditions, the function `NFTRewardsDistributor#rewardPerToken` will not update `rewardPerTokenStored[_assetHash]` due to rounding to zero

https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L77-L86

```solidity
  function rewardPerToken(bytes32 _assetHash) public view returns (uint256) {
    if (_totalSupply[_assetHash] == 0) {
      return rewardPerTokenStored[_assetHash];
    }
>>  return rewardPerTokenStored[_assetHash].add(
>>    lastTimeRewardApplicable(_assetHash).sub(lastUpdateTime[_assetHash]).mul(rewardRate[_assetHash]).mul(1e18).div(
>>      _totalSupply[_assetHash]
>>    )
>>  );
  }
```
Rewrite the formula:

$$
rewardPerTokenStored += \frac{time \times rate \times 10^{18}}{totalSupply}
$$

When the RHS is less than `1`, it will be rounded down to zero, which cause the `rewardPerTokenStored[_assetHash]` will not be updated

$$
time \times rate < \frac{totalSupply}{10^{18}}
$$

### Internal pre-conditions

- Low decimals token (`USDC`) is used as `rewardsToken` in `NFTPositionManager`.
- The condition $time \times rate < \frac{totalSupply}{10^{18}}$ is met.

### External pre-conditions

_No response_

### Attack Path

The attacker calls to `NFTRewardsDistributor#getReward(0, _assetHash)` every block.

### Impact

The rewards can not accrue in `NFTPositionManager`.

### PoC

Let's have:
- A block time of `10 seconds`
- `rewardsDuration = 7 days = 604800 seconds`
- The protocol sends the rewards of `6_048e6 USDC` to `NFTPositionManager`

The `rewardRate` would be `6_048e6 / rewardsDuration = 10_000`, so every block `10_000 * 10 = 100_000 USDC` rewards accrued. 

In this setting, the condition for the attack to be possible is 

$$
10^5 < \frac{totalSupply}{10^{18}} \Rightarrow totalSupply > 10^5 \times 10^{18}.
$$

This condition is reasonable for an ERC20 token with 18 decimals.

When the above condition is met, the attacker calls to `NFTRewardsDistributor#getReward(0, _assetHash)` every block to cause the rewards to not accrue.

### Mitigation

Convert the rewards to `wad`

`NFTRewardsDistributor.sol`

```diff
  function notifyRewardAmount(uint256 reward, address pool, address asset, bool isDebt) external onlyRole(REWARDS_ALLOCATOR_ROLE) {
    rewardsToken.transferFrom(msg.sender, address(this), reward);
+    // convert `reward` to `wad`
    bytes32 _assetHash = assetHash(pool, asset, isDebt);
    _updateReward(0, _assetHash);

    if (block.timestamp >= periodFinish[_assetHash]) {
      rewardRate[_assetHash] = reward.div(rewardsDuration);
    } else {
      uint256 remaining = periodFinish[_assetHash].sub(block.timestamp);
      uint256 leftover = remaining.mul(rewardRate[_assetHash]);
      rewardRate[_assetHash] = reward.add(leftover).div(rewardsDuration);
    }

    // Ensure the provided reward amount is not more than the balance in the contract.
    // This keeps the reward rate in the right range, preventing overflows due to
    // very high values of rewardRate in the earned and rewardsPerToken functions;
    // Reward + leftover must be less than 2^256 / 10^18 to avoid overflow.
    uint256 balance = rewardsToken.balanceOf(address(this));
+    // convert `balance` to `wad`
    require(rewardRate[_assetHash] <= balance.div(rewardsDuration), 'Provided reward too high');

    lastUpdateTime[_assetHash] = block.timestamp;
    periodFinish[_assetHash] = block.timestamp.add(rewardsDuration);
    emit RewardAdded(_assetHash, reward);
  }

  function _updateReward(uint256 _tokenId, bytes32 _assetHash) internal {
    rewardPerTokenStored[_assetHash] = rewardPerToken(_assetHash);
    lastUpdateTime[_assetHash] = lastTimeRewardApplicable(_assetHash);
    if (_tokenId != 0) {
      rewards[_tokenId][_assetHash] = earned(_tokenId, _assetHash);
      userRewardPerTokenPaid[_tokenId][_assetHash] = rewardPerTokenStored[_assetHash];
    }
  }

  function getReward(uint256 tokenId, bytes32 _assetHash) public nonReentrant returns (uint256 reward) {
    _updateReward(tokenId, _assetHash);
    reward = rewards[tokenId][_assetHash];
    if (reward > 0) {
      rewards[tokenId][_assetHash] = 0;
      rewardsToken.transfer(ownerOf(tokenId), reward);
+      // convert the reward back to the original decimal
      emit RewardPaid(tokenId, ownerOf(tokenId), reward);
    }
  }
```