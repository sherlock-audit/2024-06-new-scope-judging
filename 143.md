Smooth Zinc Puppy

High

# Malicious actors can execute sandwich attacks during market addition with existing funds

### Summary

The immediate addition of assets from a new and re-added market with existing assets in vault's position will cause a significant financial loss for existing vault users as attackers will execute a sandwich attack to profit from the asset-share ratio changes.

### Root Cause

The vulnerability stems from the immediate update of total assets when adding or re-adding a market with existing assets in the vault's position. This occurs in the _setCap method called by acceptCap:
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L85-L105

```solidity
        withdrawQueue.push(pool);

        if (withdrawQueue.length > MAX_QUEUE_LENGTH) revert CuratedErrorsLib.MaxQueueLengthExceeded();

        marketConfig.enabled = true;

        // Take into account assets of the new market without applying a fee.
        pool.forceUpdateReserve(asset());
        uint256 supplyAssets = pool.supplyAssets(asset(), positionId);
        _updateLastTotalAssets(lastTotalAssets + supplyAssets);
```

This immediate update to assets is also reflected in the totalAssets() function, which sums the balance of all markets in the withdraw queue
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVault.sol#L368
```solidity
  function totalAssets() public view override returns (uint256 assets) {
    for (uint256 i; i < withdrawQueue.length; ++i) {
      assets += withdrawQueue[i].getBalanceByPosition(asset(), positionId);
    }
  }
```

The totalAssets() value is then used in share-to-asset conversions:
```solidity
 function _accruedFeeShares() internal view returns (uint256 feeShares, uint256 newTotalAssets) {
    newTotalAssets = totalAssets();
```
```solidity
    assets = _convertToAssetsWithTotals(shares, totalSupply(), newTotalAssets, MathUpgradeable.Rounding.Up);
```

This mechanism allows an attacker to observe the acceptCap transaction and execute a sandwich attack:

Deposit assets to receive X shares for Y assets before the market addition.
After the market addition increases total assets, withdraw the same X shares to receive Y + Δ assets, where Δ is determined by the new asset-to-share ratio.

### Internal pre-conditions

1. Admin needs to call acceptCap() to add a market
2. The new/ re-added market needs to have a non-zero supplyAssets value in vault's position

### External pre-conditions

_No response_

### Attack Path

1. Attacker calls deposit() just before a market with existing position for the vault is added
2. Admin calls acceptCap() to add the market with existing funds for the vault
3. Assets of the vault are immediately increased by the amount of assets in the position for the added market, as the market is added to the withdraw queue and total assets take into account assets from all markets present in the withdraw queue
4. Attacker calls withdraw() to remove their recently deposited funds
5. Attacker receives more assets than initially deposited due to the increased asset to share ratio.

### Impact

The existing vault users suffer a loss proportional to the size of the new/ re-added market's assets relative to the total vault assets before the addition. The attacker gains this difference at the expense of other users.


**Isn't it impossible to add a market with existing funds?**
A: No, it's actually possible and even anticipated in two scenarios:
Reintegrating a previously removed market with leftover funds, e.g. A market removed due to an issue, but not all funds were withdrawn.
Adding a new market that received donations to the vaults position directly via the pool contract.

The contract specifically accounts for these cases by not charging fees on these pre-existing funds, as shown by the comment in the code `// Take into account assets of the new market without applying a fee.`

**Why is this vulnerability critical?**
A: It's critical because:

- It directly risks funds belonging to existing users which were lost when a market had to be removed with leftover funds.
- Even donations loss can be considered as a loss for existing users.


### PoC

_No response_

### Mitigation

In case of adding a market with existing funds, consider gradual unlocking of assets over a period of time.