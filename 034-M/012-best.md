Bright Tan Gibbon

Medium

# Updating the Configurator on the PoolFactory causes that all existing pools won't be able to update their Reserves Configurations.

### Summary

The PoolFactory contract is able to configure a specific contract as the Configurator contract where all the deployed [pools will configure their Pool's Admin roles.](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/manager/PoolConfigurator.sol#L37-L56)
The Configurator contract is in charge to validate that the callers are the authorized accounts that were granted the corresponding roles to make updates on the pools.
  - Each Pool has a set of admins configured on the Configurator contract. And when attempting to update a reserve's configuration, [the Pool validates that the caller is the Configurator contract that is setup on the Factory.](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/Pool.sol#L156)

The problem comes from the fact that when the [Configurator contract is updated on the PoolFactory](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/PoolFactory.sol#L98-L102):
1. All the Role's configurations for all the existing pools are configured on the Old Configurator.
2. The new Configurator contract has not knowledge of any of the existing role's configurations for the existing pools.

As explained in detail on the Attack Path and PoC sections, when the Configurator is updated on the PoolFactory, all existing pools will no longer able to update their reserves configurations, completely breaking the capability of managing the pool & reserves parameters to adapt to market activity.

### Root Cause

When updating the Reserves Configurations on the Pools, the logic [validates if the caller is the configurator that is setup on the Factory](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/Pool.sol#L156), regardless if that configurator is the one that was setup for the Pool when it was created

### Internal pre-conditions

[Configurator contract is updated on the PoolFactory to point to a new Configurator.](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/PoolFactory.sol#L98-L102)

### External pre-conditions

None.

### Attack Path

1. Configurator is updated on the PoolFactory.
2. New Pools are deployed, and the Pool's roles are setup on the new Configurator contract.
3. An existing pool requires to make some adjustments to its reserve's configurations. Let's say that a reserve needs to be frozen.
3.1 An account that was granted the RiskAdmin role calls the old configurator (The one that was used when the Pool was deployed).
  - This call will fail because the Old Configurator is not anymore the same configurator that is set on the Factory.
3.2 The same account now tries to call the new configurator to freeze the reserve.
  - This call will also fail, but now because the Pool has not been configured its roles on the new Configurator contract.

4. As a result, the Pool's admins are unable to freeze the reserve.


### Impact

Existing pools won't be able to update their reserves configurations, completely breaking the capability of managing the pool & reserves parameters to adapt to market activity.

### PoC

Add the below PoC on the [`PoolSupplyTests.t.sol`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/test/forge/core/pool/PoolSupplyTests.t.sol) test file
```solidity
function test_PoC_ChangingConfiguratorBreaksExistingPools() external {
  IPool pool1 = pool;
  configurator.setSupplyCap(IPool(address(pool1)), address(tokenA), 100);
  PoolConfigurator configurator2 = new PoolConfigurator(address(poolFactory));

  //@audit => A new configurator is setup on the PoolFactory contract
  poolFactory.setConfigurator(address(configurator2));

  //@audit => Deploy a new Pool and Validate the new configurator works on it
  poolFactory.createPool(_basicPoolInitParams());
  IPool poolAddr2 = poolFactory.pools(1);
  IPool pool2 = IPool(address(poolAddr2));
  configurator2.setSupplyCap(IPool(address(pool2)), address(tokenA), 100);

  //@audit => Attemping to set the supply cap for pool1 on the new configurator
  vm.expectRevert(bytes('not risk or pool admin'));
  configurator2.setSupplyCap(IPool(address(pool1)), address(tokenA), 150);

  //@audit => Attemping to set the supply cap for pool1 on the old configurator
  vm.expectRevert(bytes('only configurator'));
  configurator.setSupplyCap(IPool(address(pool1)), address(tokenA), 150);
}
```

Run the previous PoC with the next command: `forge test --match-test test_PoC_ChangingConfiguratorBreaksExistingPools -vvvv`

### Mitigation

When each pool is deployed, save the address of the current configurator on the pool's storage.
- In this way, each pool will have the address of the configurator contract where the pool's roles were configured, and it won't need to pull the configurator's address from the Factory.