Joyous Cedar Tortoise

High

# A malicious user can re-allocate vault funds into a pool that is about to experience bad debt to cause significant loss for the vault

### Summary

An attacker can use flash loans to re-allocate the funds of any vault into whichever pools they choose, this is best described in a set of steps.

Assume a vault deposits into pools A, B and C. Assume each pool has a cap of 60 ETH. The deposit queue is [A,B,C]. The withdraw queue is [A,B,C]. Note that all these numbers and orders are 100% arbitrary, the attack can be performed for any variation or caps/order.

Currently the allocations are as follows:

Pool A has 20 ETH deposited
Pool B has 20 ETH deposited
Pool C has 20 ETH deposited

Here is how the attacker can re-allocate all the funds into Pool C:

1. Attacker takes a flash loan of 120 ETH
2. Attacker [deposits](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVault.sol#L322) 40 ETH into the vault, this will fill the cap for Pool A
3. Attacker deposits 40 ETH into the vault, this will fill the cap for Pool B
4. Attacker deposits 40 ETH into the vault, this will fill the cap for Pool C
5. Attacker withdraws 120 ETH, this will pull 60 from pool A and 60 from pool B
6. Attacker repays the 120 ETH flash loan (no fees since it was from balancer/maker)

Here is the allocation after the attack:

Pool A has 0 ETH deposited
Pool B has 0 ETH deposited
Pool C has 60 ETH deposited

The issue occurs when a pool is about to be in bad debt, an attacker can re-allocate all the vault's funds into the pool that will experience bad debt to cause a severe loss to the depositors.

### Root Cause

Anybody can use flash loans to arbitrarily reallocate vault allocations.

### Internal pre-conditions

Any pool supplied by vault is about to experience bad debt

### External pre-conditions

_No response_

### Attack Path

1. Attacker is observing the mempool looking for any pool that is deposited to from a vault, that has accumulated bad debt BUT not realised it yet 
2. Attacker re-allocates the vault’s funds into the bad debt pool using flash loans
3. When the bad debt is realised the vault depositors will experience a huge loss

### Impact

Vault depositors experience a severe loss since their funds were re allocated into a pool that experienced bad debt

Another point to note is that bad debt is not socialised in the protocol, this makes the impact even more severe if other pool suppliers withdraw first leaving the vault depositors with nothing

### PoC

_No response_

### Mitigation

_No response_