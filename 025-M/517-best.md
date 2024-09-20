Long Coffee Cat

High

# Assets can be locked on NFTPositionManager balance due to rounding violating payback conditions

### Summary

The payback condition of a fully cleared debt can be periodically violated as rounding involved does not guarantee perfect amounts match

### Root Cause

`currentDebtBalance == 0` required as a payback condition can not be always achieved:

[NFTPositionManagerSetters.sol#L127-L129](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L127-L129)

```solidity
    if (currentDebtBalance == 0 && repaid.assets < params.amount) {
>>    asset.safeTransfer(msg.sender, params.amount - repaid.assets);
    }
```

[ReserveSuppliesConfiguration.sol#L43-L45](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/ReserveSuppliesConfiguration.sol#L43-L45)

```solidity
  function getDebtBalance(DataTypes.ReserveSupplies storage self, uint256 index) internal view returns (uint256 debt) {
>>  debt = self.debtShares.rayMul(index);
  }
```

[PositionBalanceConfiguration.sol#L107-L118](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L107-L118)

```solidity
  function repayDebt(
    DataTypes.PositionBalance storage self,
    DataTypes.ReserveSupplies storage supply,
    uint256 amount,
    uint128 index
  ) internal returns (uint256 sharesBurnt) {
>>  sharesBurnt = amount.rayDiv(index);
    require(sharesBurnt != 0, PoolErrorsLib.INVALID_BURN_AMOUNT);
    self.lastDebtLiquidtyIndex = index;
    self.debtShares -= sharesBurnt;
    supply.debtShares -= sharesBurnt;
  }
```

So, it is `a.rayMul(index).rayDiv(index) = (a.rayMul(index) * RAY + index / 2) / index = (((a * index + HALF_RAY) / RAY) * RAY + index / 2) / index` per `rayMul` and `rayDiv` definitions:

[WadRayMath.sol#L65-L95](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/utils/WadRayMath.sol#L65-L95)

```solidity
  /**
   * @notice Multiplies two ray, rounding half up to the nearest ray
   * @dev assembly optimized for improved gas savings, see https://twitter.com/transmissions11/status/1451131036377571328
   * @param a Ray
   * @param b Ray
   * @return c = a raymul b
   */
  function rayMul(uint256 a, uint256 b) internal pure returns (uint256 c) {
    // to avoid overflow, a <= (type(uint256).max - HALF_RAY) / b
    assembly {
      if iszero(or(iszero(b), iszero(gt(a, div(sub(not(0), HALF_RAY), b))))) { revert(0, 0) }

      c := div(add(mul(a, b), HALF_RAY), RAY)
    }
  }

  /**
   * @notice Divides two ray, rounding half up to the nearest ray
   * @dev assembly optimized for improved gas savings, see https://twitter.com/transmissions11/status/1451131036377571328
   * @param a Ray
   * @param b Ray
   * @return c = a raydiv b
   */
  function rayDiv(uint256 a, uint256 b) internal pure returns (uint256 c) {
    // to avoid overflow, a <= (type(uint256).max - halfB) / RAY
    assembly {
      if or(iszero(b), iszero(iszero(gt(a, div(sub(not(0), div(b, 2)), RAY))))) { revert(0, 0) }

      c := div(add(mul(a, RAY), div(b, 2)), b)
    }
  }
```

This can round the end figure, so user debt shares won't be zeroed

### Internal pre-conditions

Amounts are such that rounding produces non-full amounts match no matter how big the user supplied funds are

### External pre-conditions

A user repays a loan and provides extra funds in order to clear their position fully

### Attack Path

No attack is needed, logic malfunction

### Impact

User funds are frozen within the contract. The amount, `params.amount - repaid.assets`, can be arbitrary large

### PoC

Asset is `USDC`, index is `0.49e27`, user has `10e6 + 1` debt shares.

Then `((((10e6*1e6+1) * 0.49e27 + 0.5e27) // 1e27) * 1e27 + 0.49e27 // 2) // 0.49e27 = 10e6` (python notation), i.e. in this case `a.rayMul(index).rayDiv(index) = a - 1`, user shares aren't cleared in full. Arbitrary amount of extra funds supplied will be frozen with the contract as no payback happens

### Mitigation

Consider introducing the flag for clearing the debt fully and setting the debt to zero directly in this case