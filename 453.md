Smooth Carbon Narwhal

Medium

# The supplyQueue and withdrawQueue are updated incorrectly in the curated vault

## Summary
In the `curated vault`, `allocators` can update both the `supplyQueue` and `withdrawQueue` as needed. 
However, issues arise when the length of the new `queue` is shorter than the old `queue`.
## Vulnerability Detail
In the `setSupplyQueue` function, the `supplyQueue` is updated with the new `queue` on `line 187`. 
```solidity
function setSupplyQueue(IPool[] calldata newSupplyQueue) external onlyAllocator {
  uint256 length = newSupplyQueue.length;

  if (length > MAX_QUEUE_LENGTH) revert CuratedErrorsLib.MaxQueueLengthExceeded();
  for (uint256 i; i < length; ++i) {
    IERC20(asset()).forceApprove(address(newSupplyQueue[i]), type(uint256).max);
    if (config[newSupplyQueue[i]].cap == 0) revert CuratedErrorsLib.UnauthorizedMarket(newSupplyQueue[i]);
  }

187:  supplyQueue = newSupplyQueue;
}
```
Similarly, in the `updateWithdrawQueue` function, the `withdrawQueue` is updated on `line 227`.
```solidity
function updateWithdrawQueue(uint256[] calldata indexes) external onlyAllocator {
227:  withdrawQueue = newWithdrawQueue;
}
```
However, there is a known issue when copying `arrays` in `Solidity`. 
For example, if array `a` is `[1, 2, 3, 4, 5]` and array `b` is `[3, 3, 3]`, attempting to copy `b` to `a` using `a = b` results in `a` being `[3, 3, 3, 4, 5]` instead of the expected `[3, 3, 3]`.
## Impact
Since the `supplyQueue` and `withdrawQueue` play a critical role in the `curated vault`, ensuring they are updated correctly is essential.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVault.sol#L187
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVault.sol#L227
## Tool used

Manual Review

## Recommendation
Add elements to an `array` using the `push` method.