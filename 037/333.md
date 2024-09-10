Long Coffee Cat

Medium

# CuratedVault cannot remove pool's allowances

### Summary

Lack of allowance removal enables rogue pools to steal `asset()` balance of the Vault, i.e. there is no pool integration defense mechanics, infinite allowances are permanent once set

### Root Cause

There are infinite approval setups on supply queue update and on supply allocation, while there is no allowance removal at all:

[CuratedVault.sol#L267-L268](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVault.sol#L267-L268) and [CuratedVault.sol#L177-L189](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVault.sol#L177-L189)

```solidity
  function setSupplyQueue(IPool[] calldata newSupplyQueue) external onlyAllocator {
    uint256 length = newSupplyQueue.length;

    if (length > MAX_QUEUE_LENGTH) revert CuratedErrorsLib.MaxQueueLengthExceeded();

    for (uint256 i; i < length; ++i) {
>>    IERC20(asset()).forceApprove(address(newSupplyQueue[i]), type(uint256).max);
      if (config[newSupplyQueue[i]].cap == 0) revert CuratedErrorsLib.UnauthorizedMarket(newSupplyQueue[i]);
    }

    supplyQueue = newSupplyQueue;
    emit CuratedEventsLib.SetSupplyQueue(_msgSender(), newSupplyQueue);
  }
```

There are no guarantees that all pools have the implementation provided in the current snapshot, CuratedVault only requires that these contracts follow `IPool` interface

### Internal pre-conditions

Pool contract is added to the Vault's `supplyQueue` or allocated during `reallocate`. Vault's `asset()` balance isn't empty

### External pre-conditions

Pool contract is jeopardized, so it can be utilized to use the allowance directly anyhow on attacker's benefit

### Attack Path

After supply queue addition or supply allocation to the pool's contract it is attacked, then permanent allowance is used to steal Vault's balance

### Impact

Vault's `asset()` balance can be fully stolen

### PoC

Not needed, it's a direct use of token allowance

### Mitigation

Consider resetting it to zero when pool's cap is reduced to zero, e.g.:

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVault.sol#L148-L155

```diff
  function submitCap(IPool pool, uint256 newSupplyCap) external onlyCuratorRole {
    if (pendingCap[pool].validAt != 0) revert CuratedErrorsLib.AlreadyPending();
    if (config[pool].removableAt != 0) revert CuratedErrorsLib.PendingRemoval();
    uint256 supplyCap = config[pool].cap;
    if (newSupplyCap == supplyCap) revert CuratedErrorsLib.AlreadySet();

    if (newSupplyCap < supplyCap) {
      _setCap(pool, newSupplyCap.toUint184());
+     if (newSupplyCap == 0) IERC20(asset()).forceApprove(address(pool), 0);
```

Also, to unify the logic and reduce overhead, consider moving `IERC20(asset()).forceApprove(address(pool), type(uint256).max)` operations to the moment when pool's cap is increased, e.g. to `acceptCap()`:

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVault.sol#L306-L309

```diff
  function acceptCap(IPool pool) external afterTimelock(pendingCap[pool].validAt) {
    // Safe "unchecked" cast because pendingCap <= type(uint184).max.
    _setCap(pool, uint184(pendingCap[pool].value));
+   IERC20(asset()).forceApprove(address(pool), type(uint256).max);
  }
```