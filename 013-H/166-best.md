Massive Nylon Vulture

High

# `GenericLogic.sol` contract assumes all price feeds has the same decimals but is a wrong assumption that leads to an incorrect health factor math.

### Summary

Mixing price feeds decimals when doing the calculation of  `totalCollateralInBaseCurrency` and `totalDebtInBaseCurrency` will cause an incorrect `healthFactor` affecting important operations of the protocol such as `liquidation`, `borrow` and `withdraw`. `GenericLogic.sol` contract assumes all price feeds have the same decimals but is a wrong assumption as is demonstrated in the Root cause section.

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/GenericLogic.sol#L106-L128

### Root Cause

`GenericLogic.sol:69 calculateUserAccountData` calculates the `totalCollateralInBaseCurrency` and the `totalDebtInBaseCurrency` doing a sum of all the differents reserve assets as collateral or debt in base currency but the problem is a wrong assumption that all chainlink price feeds has the same decimals. Most USD price feeds has 8 decimals but for example [AMPL / USD](https://etherscan.io/address/0xe20CA8D7546932360e37E9D72c1a47334af57706) feed has 18 decimals. So `totalCollateralInBaseCurrency` and the `totalDebtInBaseCurrency` will be incorrect because `calculateUserAccountData` will sum asset prices with different decimals leading to a wrong calculation of the health factor and incorrect function of many protocol operations. 

### Internal pre-conditions

1. Price feeds for collateral or debt assets in a given position needs to have different decimals.


### External pre-conditions

None

### Attack Path

Is a wrong assumption proven by example


### Impact

1. Liquidation: Mixing Price decimals lead to incorrect calculation of the `healthFactor` that is a result of wrong `totalCollateralInBaseCurrency` and the `totalDebtInBaseCurrency`.
2. Borrow: Wrong `healthFactor` also affects borrowing when doing validations to make sure that the position is not liquiditable.
3. Withdraw: Also uses `healthFactor` via `ValidationLofic::validateHFAndLtv`
4. executeUseReserveAsCollateral: Also uses `healthFactor` via `ValidationLofic::validateHFAndLtv`
5. Any other operation that uses the health factor.


### PoC

none

### Mitigation

There are 2 possible solution:

1. Some protocols enforce 8 decimals when assigning an oracle to an asset or reject the operation. (easy, simple, secure, not flexible)
2. Use AggregatorV3Interface::decimals to normalize to N decimals the price making sure that the precision loss is on the correct side. (flexible)

