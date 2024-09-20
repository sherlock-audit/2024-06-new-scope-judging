Clever Ebony Halibut

Medium

# Immutability of Oracles in Pools Leading to Potential Pool Bricking

## Summary
The ZeroLend protocol lacks functionality to update oracle addresses after pool initialization, potentially leading to pool bricking if an oracle becomes deprecated or malfunctions. This immutability can result in locked user assets, unusable pools, and increased vulnerability to exploits.

## Vulnerability Detail
In the current implementation, oracles are set during pool initialization and cannot be changed afterwards. This is evident from the following code analysis:

1. In `PoolFactory.sol`, pools are created and initialized in the `createPool` function:
```solidity
function createPool(DataTypes.InitPoolParams memory params) external returns (IPool pool) {
// ... (pool creation logic)
pool.initialize(params);
configurator.initRoles(IPool(address(pool)), params.admins, params.emergencyAdmins, params.riskAdmins);
}
```
The `initialize` function is called with `params`, that includes the oracle address for the designated reserves in the pool.
The pool and the pool Configurators however doesn't include any function to update the oracle address.

In the `PoolConfigurator.sol` contract, the protocol implements multiple functions to allow pool admins to configure risk on the pool for example by changing interestrate model using  `setReserveInterestRateStrategyAddress` but it **doesn't implement any function to update the oracle address.**

## Impact
The inability to update or replace an oracle in the ZeroLend protocol can lead to a situation where the liquidity pools relying on that oracle can no longer operate correctly, either due to an oracle being malfunctioning or deprecated.    

This can prevent users from performing any operations such as deposits, withdrawals, or loan repayments, thereby locking their assets in the protocol and rendering the pools effectively useless. This could also expose poolds to potential exploits and loss of funds in case an immutable oracle malfunctions.

## Code Snippet
- https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/manager/PoolConfigurator.sol#L27
## Tool used

Manual Review

## Recommendation
Consider implementing a function in the PoolConfigurator contract that allows to update the oracle address of a reserve in a pool by the pool admins.