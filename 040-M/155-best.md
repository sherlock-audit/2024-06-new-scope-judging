Rapid Onyx Meerkat

High

# Supplying assets via the pool can result in the user losing rewards.

### Summary

When a user supplies assets through the `NFTPositionManager` contract, the `_updateReward` function is called to record the rewards for a specific `tokenId`. However, since anyone can permissionlessly invoke the `Pool.sol::supply()` function to supply assets for a specific `tokenId`, the rewards are not updated, resulting in the user losing potential rewards.

### Root Cause

When user supply assets through the `NFTPositionManager` contract , [NFTRewardsDistributor.sol::_updateReward](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L151-L158)  is called.
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
[NFTRewardsDistributor.sol::earned](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L98-L102) the earned amount is based on user's balance.
```solidity
  function earned(uint256 tokenId, bytes32 _assetHash) public view returns (uint256) {
    return _balances[tokenId][_assetHash].mul(rewardPerToken(_assetHash).sub(userRewardPerTokenPaid[tokenId][_assetHash])).div(1e18).add(
      rewards[tokenId][_assetHash]
    );
  }
```
however anyone can permissionlessly invoke the  [Pool.sol::supply](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/Pool.sol#L67-L75)  function to supply assets for a specific `tokenId`
```solidity
  function supply(
    address asset,
    address to, //@audit not in track.
    uint256 amount,
    uint256 index,//@audit-info is tokenId when called from NFT.
    DataTypes.ExtraData memory data
  ) public returns (DataTypes.SharesType memory) {//@audit-info postionId is consist of, user-index-tokenId.
    return _supply(asset, amount, to.getPositionId(index), data);
  }
```
`_updateReward` is not called 

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

user lost of reward token

### PoC

```solidity
  function testErrorTrackBalance() public {
    vm.warp(1641070800);

    //alice mint position
    uint256 mintAmount = 100 ether;
    DataTypes.ExtraData memory data = DataTypes.ExtraData(bytes(''), bytes(''));

    _mintAndApprove(address(this), tokenA, mintAmount, address(pool));

    vm.startPrank(alice);
    nftPositionManager.mint(address(pool));
    vm.stopPrank();

    //supply asset hash.
    bytes32 assetHash = keccak256(abi.encode(pool, address(tokenA), false));

    //before.
    assertEq(nftPositionManager.balanceOfByAssetHash(1,assetHash),0);

    //supply for alice's postion via pool.
    pool.supply(address(tokenA), address(nftPositionManager), 10e18, 1, data);

    assertEq(nftPositionManager.balanceOfByAssetHash(1,assetHash),0);

    //alice try to withdraw assets via nftPositionManager
    vm.startPrank(alice);
    INFTPositionManager.AssetOperationParams memory params = INFTPositionManager.AssetOperationParams(address(tokenA), alice, 10e18, 1, data);
    nftPositionManager.withdraw(params);

    assertEq(nftPositionManager.balanceOfByAssetHash(1,assetHash),0);

    assertEq(tokenA.balanceOf(address(alice)),10e18);
  }
```

From above test we can see the balance of alice is zero during the whole process.

### Mitigation

```diff
@@ -71,7 +71,7 @@ contract Pool is PoolSetters {
     uint256 index,
     DataTypes.ExtraData memory data
   ) public returns (DataTypes.SharesType memory) {
-    return _supply(asset, amount, to.getPositionId(index), data);
+    return _supply(asset, amount, msg.sender.getPositionId(index), data);
   }
```