Smooth Carbon Narwhal

Medium

# The lastTotalAsset should be updated when pools are removed from the withdrawQueue

## Summary
`Pools` can be removed from the `withdrawQueue` for various reasons, and in some cases, `assets` may still be `deposited` in these `pools`. 
When this happens, it's important to update `lastTotalAsset` because it could be larger than the new `total assets`. 
Additionally, `fees` should be minted before removing `pools` from the `withdrawQueue`.
## Vulnerability Detail
In the `_accrueFee` function, the `_accruedFeeShares` function is called to mint `fees` to the `fee recipient`. 
```solidity
function _accrueFee() internal returns (uint256 newTotalAssets) {
  uint256 feeShares;
  (feeShares, newTotalAssets) = _accruedFeeShares();
  if (feeShares != 0) _mint(feeRecipient, feeShares);
  emit CuratedEventsLib.AccrueInterest(newTotalAssets, feeShares);
}
```
The `_accruedFeeShares` function calculates the `increase` in `total assets` and mints a portion of this `increase` as `fees` to the `recipient`. 
On line 188, the increase is calculated as the difference between the current `total assets` and the last recorded `lastTotalAsset`.
```solidity
function _accruedFeeShares() internal view returns (uint256 feeShares, uint256 newTotalAssets) {
  newTotalAssets = totalAssets();

188:  uint256 totalInterest = newTotalAssets.zeroFloorSub(lastTotalAssets);
}
```
`Allocators` can remove `pools` from the `withdrawQueue` for several reasons. 
However, before updating the `withdrawQueue`, we currently don't mint the `fees` that have accumulated so far. 
Furthermore, on `line 216`, if there are still supplied `assets` in the removed `pools`, we didn't  update `lastTotalAssets`.
```solidity
function updateWithdrawQueue(uint256[] calldata indexes) external onlyAllocator {
  uint256 newLength = indexes.length;
  uint256 currLength = withdrawQueue.length;

  bool[] memory seen = new bool[](currLength);
  IPool[] memory newWithdrawQueue = new IPool[](newLength);

  for (uint256 i; i < newLength; ++i) {
    uint256 prevIndex = indexes[i];
    IPool pool = withdrawQueue[prevIndex];
    if (seen[prevIndex]) revert CuratedErrorsLib.DuplicateMarket(pool);
    seen[prevIndex] = true;
    newWithdrawQueue[i] = pool;
  }

  for (uint256 i; i < currLength; ++i) {
    if (!seen[i]) {
      IPool pool = withdrawQueue[i];
      if (config[pool].cap != 0) revert CuratedErrorsLib.InvalidMarketRemovalNonZeroCap(pool);
      if (pendingCap[pool].validAt != 0) revert CuratedErrorsLib.PendingCap(pool);
216:      if (pool.supplyShares(asset(), positionId) != 0) {
        if (config[pool].removableAt == 0) revert CuratedErrorsLib.InvalidMarketRemovalNonZeroSupply(pool);
        if (block.timestamp < config[pool].removableAt) {
          revert CuratedErrorsLib.InvalidMarketRemovalTimelockNotElapsed(pool);
        }
      }

      delete config[pool];
    }
  }

  withdrawQueue = newWithdrawQueue;
}
```
As a result, during the next call to `_accrueFee`, the current `total assets` will appear lower than `lastTotalAssets` because the `assets` `deposited` in the removed `pools` are no longer included in the `total` but remain in `lastTotalAssets`. 
Consequently, the `fee recipient` won't receive the correct `fees`.
## Impact
Since this protocol is designed to mint `fee shares` to the `fee recipient`, this process must work correctly as intended.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L178
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultGetters.sol#L188
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVault.sol#L216

## Tool used

Manual Review

## Recommendation
```solidity
function updateWithdrawQueue(uint256[] calldata indexes) external onlyAllocator {
+  _updateLastTotalAssets(_accrueFee());

  uint256 newLength = indexes.length;
  uint256 currLength = withdrawQueue.length;

  bool[] memory seen = new bool[](currLength);
  IPool[] memory newWithdrawQueue = new IPool[](newLength);

  for (uint256 i; i < newLength; ++i) {
    uint256 prevIndex = indexes[i];
    IPool pool = withdrawQueue[prevIndex];
    if (seen[prevIndex]) revert CuratedErrorsLib.DuplicateMarket(pool);
    seen[prevIndex] = true;
    newWithdrawQueue[i] = pool;
  }

  for (uint256 i; i < currLength; ++i) {
    if (!seen[i]) {
      IPool pool = withdrawQueue[i];
      if (config[pool].cap != 0) revert CuratedErrorsLib.InvalidMarketRemovalNonZeroCap(pool);
      if (pendingCap[pool].validAt != 0) revert CuratedErrorsLib.PendingCap(pool);
      if (pool.supplyShares(asset(), positionId) != 0) {
        if (config[pool].removableAt == 0) revert CuratedErrorsLib.InvalidMarketRemovalNonZeroSupply(pool);
        if (block.timestamp < config[pool].removableAt) {
          revert CuratedErrorsLib.InvalidMarketRemovalTimelockNotElapsed(pool);
        }
      }

      delete config[pool];
    }
  }

+  _updateLastTotalAssets(_accrueFee());

  withdrawQueue = newWithdrawQueue;
}
```