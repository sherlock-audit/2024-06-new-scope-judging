Acrobatic Banana Poodle

High

# Function `executeMintToTreasury` will incorrectly reduce the `supplyShares`, therefore prevent the last users  from withdrawing

### Summary

The incorrect reduction in `PoolLogic::executeMintToTreasury` will cause failure of some (likely to be the last) user's withdrawal, and the fund will be locked.

### Root Cause

The `totalSupply.supplyShares` is supposed to be the sum of `balances.supplyShares`, as they are always updated in tandem: 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L94-L95
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L48-L49

If this is not held, then some users will not be able to withdraw their collateral, as the `totalSupply.supplyShares` will underflow and revert. 

However in the `PoolLogic::executeMintToTreasury` updates the `totalSupply.supplyShares` without updating any user's balance. It is because it incorrectly assumes that there is share to be burned, even though the accrued amount was never really minted to the treasury (in that the treasury's share balance was not added). If it is so, that share should be burned from the treasury.

Also, for example, when premium is added via flashloan, the premium is counted as underlyingBalance: https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/FlashLoanLogic.sol#L118

Therefore, when the underlying asset is transferred out via `executeMintToTreasury`, the `underlyingBalance` should be updated accordingly.


### Internal pre-conditions

Non-zero `accruedToTreasuryShares`


### External pre-conditions

_No response_

### Attack Path

1. Anybody calls `withdraw` or `withdrawSimple`, it will reduce the `asset`'s `totalSupply.supplyShares` incorrectly.

### impact

The last user(s) who is trying to withdraw will fail, and their fund will be locked


### PoC

_No response_

### Mitigation

Suggestion of mitigation:


```solidity
// PoolLogic.sol
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

      IERC20(asset).safeTransfer(treasury, amountToMint);
-     totalSupply.supplyShares -= accruedToTreasuryShares;
+     totalSupply.underlyingBalance -= amountToMint;

      emit PoolEventsLib.MintedToTreasury(asset, amountToMint);
    }
  }
```

