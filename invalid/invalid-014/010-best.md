Dancing Cherry Yak

High

# Insufficient Pool Verification Allows Malicious Curator to Drain Vault Funds via Fake Pools

### Summary

The `CuratedVault::submitCap` function allows the addition of pools that have not been validated by the factory, which could enable a malicious curator to add a fraudulent pool. This malicious pool could be used to drain funds allocated to it through unauthorized withdrawals. While there is a mandatory waiting period before a newly added pool is accepted, the risk of this attack persists.

### Root Cause

The `submitCap` function does not check whether the submitted pool was deployed by the factory, allowing unverified pools to be added. 

https://github.com/sherlock-audit/2024-06-new-scope/blob/a150815e6e6cae8b14a4ca5bb05d545f6a5e07ae/zerolend-one/contracts/core/vaults/CuratedVault.sol#L148

After the waiting period, the pool can be accepted without further validation.

https://github.com/sherlock-audit/2024-06-new-scope/blob/a150815e6e6cae8b14a4ca5bb05d545f6a5e07ae/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L85

### Attack Path

1. A malicious user creates a legitimate vault and adds a pool to it
2. After attracting some LP funds, the user adds a fake pool with withdrawal capabilities
3. Once this pool is accepted after the mandatory waiting period, the malicious user can allocate funds to it
4. withdraw these funds, draining the vault.

### Impact

A malicious curator can exploit this vulnerability to drain funds from the vault by introducing a fake pool that allows withdrawals.


### Mitigation

Implement a validation check in the submitCap function to ensure that only pools deployed by the factory can be submitted. The system already includes functionality to track pools deployed by the factory, so this implementation involves ensuring that the validation check is applied during the submitCap process.

https://github.com/sherlock-audit/2024-06-new-scope/blob/a150815e6e6cae8b14a4ca5bb05d545f6a5e07ae/zerolend-one/contracts/core/positions/NFTPositionManager.sol#L59