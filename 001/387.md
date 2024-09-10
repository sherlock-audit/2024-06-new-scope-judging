Sneaky Hazelnut Lizard

Medium

# After a User withdraws The interest Rate is not updated accordingly leading to the next user using an inflated index during next deposit before the rate is normalized again

### Summary

A bug in Zerolend's withdrawal mechanism causes the interest rate to not be updated when funds are transferred to the treasury during a withdrawal. This failure leads to the next user encountering an inflated interest rate when performing subsequent actions like deposit, withdrawal or borrow before the rate is normalized again. The issue arises because the liquidity in the pool drops due to the funds being transferred to the treasury, but the system fails to update the interest rate to reflect this change.

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

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

In the current implementation of Zerolend, during a **withdrawal**, the protocol transfers a portion of the funds to the **treasury**. However, it does not update the interest rate before this transfer has being done for all transfers, leading to an **inflated liquidity rate** being used by the next user, particularly for deposits. This is problematic as the next user deposits/withdraws at a rate that is incorrectly high, causing them to receive fewer shares than they should.

In comparison, Aave mints shares to the treasury, which can later withdraw this funds like any other user. 

Each withdrawal out of the contract in underlying asset **in Aave** updates the interest rate, ensuring the rates reflect the true liquidity available in the pool.

 Zerolend's approach of transferring funds directly upon every user withdrawal fails to adjust the interest rate properly, resulting in a temporary discrepancy that affects subsequent users.

#### Code Context:

In the **`executeMintToTreasury`** function, the accrued shares for the treasury are transferred, but the interest rates are not updated to account for the change in liquidity.

```solidity
function executeMintToTreasury(
    DataTypes.ReserveSupplies storage totalSupply,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    address treasury,
    address asset
  ) external {
    DataTypes.ReserveData storage reserve = reservesData[asset];

    uint256 accruedToTreasuryShares = reserve.accruedToTreasuryShares;

    if (accruedToTreasuryShares != 0) {
      reserve.accruedToTreasuryShares = 0;
      uint256 normalizedIncome = reserve.getNormalizedIncome();
      uint256 amountToMint = accruedToTreasuryShares.rayMul(normalizedIncome);

@audit>> no interest rate update before fund removal >>       IERC20(asset).safeTransfer(treasury, amountToMint);

      totalSupply.supplyShares -= accruedToTreasuryShares;

      emit PoolEventsLib.MintedToTreasury(asset, amountToMint);
    }
  }
```

As can be seen in this snippet, the funds are transferred to the treasury, but the function does not invoke any interest rate update mechanism. The liquidity in the pool decreases, but the next user's deposit will use an inflated rate due to this oversight.

#### Interest Rate Update Example (Correct Flow):

In other parts of the code, such as during withdrawals, the interest rate is properly updated when liquidity changes:

```solidity

 function executeWithdraw(
    mapping(address => DataTypes.Res

---------------------------------------
reserve.updateInterestRates(
  totalSupplies,
  cache,
  params.asset,
  IPool(params.pool).getReserveFactor(),
  0,  // No liquidity added
  params.amount,  // Liquidity taken during withdrawal
  params.position,
  params.data.interestRateData
);
```

The **`updateInterestRates`** function correctly calculates the new interest rate based on the changes in liquidity, ensuring the system uses accurate rates for subsequent operations.

#### Example of Problem:

Consider the following scenario:
- A user withdraws a portion of funds, which triggers the transfer of some assets to the treasury.
- The liquidity in the pool drops, but the interest rate is not updated.
- The next user deposits into the pool using the **inflated liquidity rate**, resulting in fewer shares being minted for them.

Since the actual liquidity is lower than the interest rate assumes, the user depositing gets fewer shares than expected.

---

### Impact

- **Incorrect Share Calculation**: Users depositing after a treasury withdrawal will receive fewer shares due to an artificially high liquidity rate than the appropriate one , leading to loss of potential value.

### PoC

_No response_

### Mitigation

The mitigation involves ensuring that the **interest rate** is properly updated **before** transferring funds to the treasury. The rate update should account for the liquidity being transferred out, ensuring the new rates reflect the actual available liquidity in the pool.

#### Suggested Fix:

In the **`executeMintToTreasury`** function, call the **`updateInterestRates`** function **before** transferring the assets to the treasury. This will ensure that the interest rate reflects the updated liquidity in the pool before the funds are moved.

##### Modified Code Example:

```solidity
function executeMintToTreasury(
    DataTypes.ReserveSupplies storage totalSupply,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    address treasury,
    address asset
  ) external {
    DataTypes.ReserveData storage reserve = reservesData[asset];

    uint256 accruedToTreasuryShares = reserve.accruedToTreasuryShares;

    if (accruedToTreasuryShares != 0) {
      reserve.accruedToTreasuryShares = 0;
      uint256 normalizedIncome = reserve.getNormalizedIncome();
      uint256 amountToMint = accruedToTreasuryShares.rayMul(normalizedIncome);

++     // Update the interest rates before transferring to the treasury
++      reserve.updateInterestRates(
++        totalSupply,
++       DataTypes.ReserveCache({}), // Supply necessary cache data
++        asset,
++       IPool(asset).getReserveFactor(),
++        0, // No liquidity added
++       amountToMint, // Liquidity taken corresponds to amount sent to treasury
++        bytes32(0), // Position details (if any)
++       new bytes(0) // Interest rate data (if any)
++      );

      IERC20(asset).safeTransfer(treasury, amountToMint);
      totalSupply.supplyShares -= accruedToTreasuryShares;

      emit PoolEventsLib.MintedToTreasury(asset, amountToMint);
    }
  }
```

In this updated version, the interest rates are recalculated to account for the **liquidity sent** to the treasury. This ensures that the **next user's deposit** uses a correctly updated interest rate.

---
