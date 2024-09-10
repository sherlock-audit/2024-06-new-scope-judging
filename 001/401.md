Sneaky Hazelnut Lizard

Medium

# Liquidation fails to update the interest Rate when liquidation funds are sent to the treasury thus the next user uses an inflated index

### Summary

A bug exists in the Zerolend liquidation process where the interest rate is not updated before transferring liquidation funds to the treasury. This omission leads to an inflated index being used by the next user when performing subsequent actions such as deposits, withdrawals, or borrowing, similar to the previously reported bug in the withdrawal function. As a result, the next user may receive fewer shares or incur an incorrect debt due to the artificially high liquidity rate.

---


### Root Cause


Examples of update rate before transferring everywhere in the protocol to maintain Rate 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L69-L81

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L125-L146

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L88-L99

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L139-L158

The same process can be observed in Aave v 3.

1. https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/SupplyLogic.sol#L130
2. https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/SupplyLogic.sol#L65
3. https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/BorrowLogic.sol#L145-L150
4.  https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/BorrowLogic.sol#L227-L232

Looking at the effect of updating rate 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L134-L182

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/periphery/ir/DefaultReserveInterestRateStrategy.sol#L98-L131

This rates are used to get the new index

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L225-L227

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L235-L237

-


### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

During the liquidation process in Zerolend, when funds are transferred to the **treasury** as a liquidation protocol fee, the interest rate in the pool is **not updated** before the transfer. This failure results in the next user's interaction with the protocol (such as a deposit, withdrawal, or loan) being calculated based on an **inflated liquidity rate**. The inflated rate causes the user to receive fewer shares than they should or be charged an incorrect interest rate.

In contrast, **Aave’s approach** ensures that the interest rate is always updated when necessary and adjusted when funds are moved outside the system. Aave achieves this by transferring the funds inside the contract in the form of **aTokens**, which track liquidity changes, and since atokens are not burnt there is no need to update the interest rate accordingly in this case. 

Zerolend, however, directly transfers funds out of the pool without recalculating the interest rate, which leads to inconsistencies in the index used by the next user.

#### Code Context:

In Zerolend's liquidation process, when a user is liquidated and the liquidation fee is sent to the treasury, the protocol transfers the funds directly without updating the interest rate.

```solidity
// Transfer fee to treasury if it is non-zero
if (vars.liquidationProtocolFeeAmount != 0) {
    uint256 liquidityIndex = collateralReserve.getNormalizedIncome();
    uint256 scaledDownLiquidationProtocolFee = vars.liquidationProtocolFeeAmount.rayDiv(liquidityIndex);
    uint256 scaledDownUserBalance = balances[params.collateralAsset][params.position].supplyShares;

    if (scaledDownLiquidationProtocolFee > scaledDownUserBalance) {
        vars.liquidationProtocolFeeAmount = scaledDownUserBalance.rayMul(liquidityIndex);
    }
@audit >> transferring underlying asset out without updating interest rate first>>>>

    IERC20(params.collateralAsset).safeTransfer(IPool(params.pool).factory().treasury(), vars.liquidationProtocolFeeAmount);
}
```

As can be seen in the code, the liquidation protocol fee is transferred to the treasury, but no interest rate update takes place **before** the transfer. This results in an incorrect liquidity rate for the next user interaction.

#### Comparison with Aave:

Aave uses **aTokens** for transfers within the protocol, and the interest rate is updated accordingly when funds are moved, ensuring that the liquidity rate and index are always accurate. In Aave’s liquidation process, the aTokens are transferred to the treasury rather than removing liquidity directly from the pool.

```solidity
vars.collateralAToken.transferOnLiquidation(
    params.user,
    vars.collateralAToken.RESERVE_TREASURY_ADDRESS(),
    vars.liquidationProtocolFeeAmount
);
```

In Aave’s implementation, the **aToken** system ensures that the liquidity and interest rates are intact based on the movement of funds and not transferring underlying assets.

---

### Impact

- **Incorrect Share Calculation**: Deposits, withdrawals, and loans after a liquidation may use an inflated liquidity rate, resulting in **fewer shares** minted for depositors or incorrect debt calculations for borrowers.
- **Protocol Inconsistency**: The protocol operates with an inaccurate interest rate after each liquidation, leading to potential financial discrepancies across user interactions.

### PoC

_No response_

### Mitigation

To address this issue, the **interest rate must be updated** before transferring any liquidation protocol fees to the treasury. This ensures that the system correctly accounts for the reduction in liquidity due to the transfer. This will be my last report here before transferring funds to the treasury also a bug was discovered before transferring. kind fix also. thank you for the great opportunity to audit your code i wish zerolend the very best in the future.

#### Suggested Fix:

In the liquidation logic, invoke the **`updateInterestRates`** function on the **collateral reserve** before transferring the funds to the treasury. This will ensure that the correct liquidity rate is applied to the pool before the funds are removed.

##### Modified Code Example:

```solidity

if (vars.liquidationProtocolFeeAmount != 0) {
    uint256 liquidityIndex = collateralReserve.getNormalizedIncome();
    uint256 scaledDownLiquidationProtocolFee = vars.liquidationProtocolFeeAmount.rayDiv(liquidityIndex);
    uint256 scaledDownUserBalance = balances[params.collateralAsset][params.position].supplyShares;

    if (scaledDownLiquidationProtocolFee > scaledDownUserBalance) {
        vars.liquidationProtocolFeeAmount = scaledDownUserBalance.rayMul(liquidityIndex);
    }

++   // Before transferring liquidation protocol fee to treasury, update the interest rates
++   collateralReserve.updateInterestRates(
++  totalSupplies,
++  collateralReserveCache,
++  params.collateralAsset,
++  IPool(params.pool).getReserveFactor(),
++  0, // No liquidity added
++   vars.liquidationProtocolFeeAmount, // Liquidity taken during liquidation
++  params.position,
++ params.data.interestRateData
++ );

++ // Now, transfer fee to treasury if it is non-zero

    IERC20(params.collateralAsset).safeTransfer(IPool(params.pool).factory().treasury(), vars.liquidationProtocolFeeAmount);
}
```

In this updated version, the interest rates are recalculated **before** the liquidation protocol fee is transferred to the treasury. This ensures that subsequent deposits, withdrawals, and loans use the correct liquidity rate and avoid discrepancies caused by an inflated index.