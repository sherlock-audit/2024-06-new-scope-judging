Careful Fleece Pike

Medium

# `BNB` rewards can not be distributed in `NFTPositionManager` when being used as `rewardsToken`

### Summary

The usage of `IERC20#transfer` instead of `SafeERC20#safeTransfer` in `NFTPositionManager` will cause `BNB` rewards can not be distributed when being used as `rewardsToken`.

### Root Cause

From the contest's `README`, the protocol is planning to support `BNB`. But, `BNB#transfer` does not return a `bool` back on success, which does not follow the `IERC20` interface.

In `NFTRewardsDistributor#getReward`, the `rewardsToken` is sent out using `IERC20#transfer`, which will cause `getReward` to revert when `BNB` is used as `rewardsToken`

https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L51

https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L93

```solidity
  function __NFTRewardsDistributor_init(
    uint256 maxBoostRequirement_,
    address staking_,
    uint256 rewardsDuration_,
    address rewardsToken_
  ) internal onlyInitializing {
    maxBoostRequirement = maxBoostRequirement_;
    stakingToken = IVotes(staking_);
>>  rewardsToken = IERC20(rewardsToken_);
    rewardsDuration = rewardsDuration_;
  }
```

```solidity
  function getReward(uint256 tokenId, bytes32 _assetHash) public nonReentrant returns (uint256 reward) {
    _updateReward(tokenId, _assetHash);
    reward = rewards[tokenId][_assetHash];
    if (reward > 0) {
      rewards[tokenId][_assetHash] = 0;
>>    rewardsToken.transfer(ownerOf(tokenId), reward);
      emit RewardPaid(tokenId, ownerOf(tokenId), reward);
    }
  }
```

### Internal pre-conditions

`BNB` is used as `rewardsToken` in `NFTPositionManager`.

### External pre-conditions

_No response_

### Attack Path

1. The protocol deploys a `NFTPositionManager` with `BNB` as `rewardsToken`.
2. The protocol sends `BNB` rewards to the contract using `NFTRewardsDistributor#notifyRewardAmount`. Note that, this function will execute successfully, because `rewardsToken.transferFrom` does not revert.
3. Users can not withdraw `BNB` rewards because `NFTRewardsDistributor#getReward` will revert.

### Impact

`BNB` rewards can not be distributed in `NFTPositionManager`.

### PoC

`forge test --match-path test/PoC/PoC.t.sol --fork-url <mainnet_rpc_url>`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import {IERC20} from '@openzeppelin/contracts/token/ERC20/IERC20.sol';
import {Test, console} from '../../lib/forge-std/src/Test.sol';

contract PoC is Test {
    address alice = makeAddr('alice');
    address bob = makeAddr('bob');

    function testPoC() public {
        IERC20 BNB = IERC20(0xB8c77482e45F1F44dE1745F52C74426C631bDD52);
        deal(address(BNB), alice, 10 ether);

        vm.prank(alice);
        BNB.transfer(bob, 5 ether); // <=== This will revert
    }
}
```

### Mitigation

Use `SafeERC20#safeTransfer, safeTransferFrom` instead of `IERC20#transfer, transferFrom` in `NFTRewardsDistributor`

```diff
+import {SafeERC20} from '@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol';

abstract contract NFTRewardsDistributor is
  ReentrancyGuardUpgradeable,
  ERC721EnumerableUpgradeable,
  AccessControlEnumerableUpgradeable,
  NFTPositionManagerGetters
{
  using SafeMath for uint256;
+ using SafeERC20 for IERC20;

  function getReward(uint256 tokenId, bytes32 _assetHash) public nonReentrant returns (uint256 reward) {
    _updateReward(tokenId, _assetHash);
    reward = rewards[tokenId][_assetHash];
    if (reward > 0) {
      rewards[tokenId][_assetHash] = 0;
-     rewardsToken.transfer(ownerOf(tokenId), reward);
+     rewardsToken.safeTransfer(ownerOf(tokenId), reward);
      emit RewardPaid(tokenId, ownerOf(tokenId), reward);
    }
  }

  function notifyRewardAmount(uint256 reward, address pool, address asset, bool isDebt) external onlyRole(REWARDS_ALLOCATOR_ROLE) {
-   rewardsToken.transferFrom(msg.sender, address(this), reward);
+   rewardsToken.safeTransferFrom(msg.sender, address(this), reward);
    ...
  }
}
```