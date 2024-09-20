Joyous Cedar Tortoise

High

# Malicious pool deployer can set a malicious interest rate contract to lock funds of vault depositors

### Summary

Once vault depositors have deposited funds into a pool, a malicious pool creator can upgrade the `interestRateStrategy` contract (using `PoolConfigurator.setReserveInterestRateStrategyAddress()` to make all calls to it revert.

As a result any function of the protocol that calls `updateInterestRates()` will revert because `updateInterestRates()` makes the following [call](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L160-L171):
```solidity
(vars.nextLiquidityRate, vars.nextBorrowRate) = IReserveInterestRateStrategy(_reserve.interestRateStrategyAddress)
      .calculateInterestRates(
      /* PARAMS */
    );
```

The main impact is that now withdrawals will revert because `executeWithdraw()` calls `updateInterestRates()` which will always revert, so the funds that vault users deposited into this pool are lost forever.

### Root Cause

Allowing the pool deployer to specify any `interestRateStrategyAddress`

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

1. Vault users deposit into the pool
2. the deployer sets their `interestRateStrategy` contract to make all calls to it revert
3. All calls to withdraw funds from the pool will revert, the vault depositors have lost their funds

### Impact

All the funds deposited to the pool from the vault will be lost

### PoC

_No response_

### Mitigation

Use protocol whitelisted interest rate calculation contracts