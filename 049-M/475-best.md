Ancient Gingerbread Rooster

Medium

# `getNormalizedDebt` returns inflated debt liquidity even if there's no debt

## Summary
The `getNormalizedDebt` function miscalculates debt liquidity, inflating values even without `debtShares`, leading to an inaccurate portrayal of debt status and potential operational discrepancies.

## Vulnerability Detail
`ReserveLogic` has `getNormalizedDebt` function that calculates expected debt liquidity of a reserve at current timepoint.

```solidity
  function getNormalizedDebt(DataTypes.ReserveData storage reserve) internal view returns (uint256) {
    uint40 timestamp = reserve.lastUpdateTimestamp;

    //solium-disable-next-line
    if (timestamp == block.timestamp) {
      //if the index was updated in the same block, no need to perform any calculation
      return reserve.borrowIndex;
    } else {
>     return MathUtils.calculateCompoundedInterest(reserve.borrowRate, timestamp).rayMul(reserve.borrowIndex);
    }
  }
```
The function erroneously inflates the liquidity of debt, including interest, regardless of the actual debt status. In contrast, the `_updateIndexes` function correctly increases liquidity only when there is a non-zero debt for the reserve, highlighting a discrepancy in how debt liquidity is calculated and updated across these functions.

```solidity
  function _updateIndexes(DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
    ...
    // Variable borrow index only gets updated if there is any variable debt.
    // cache.currBorrowRate != 0 is not a correct validation,
    // because a positive base variable rate can be stored on
    // cache.currBorrowRate, but the index should not increase
>   if (_cache.currDebtShares != 0) {
      uint256 cumulatedBorrowInterest = MathUtils.calculateCompoundedInterest(_cache.currBorrowRate, _cache.reserveLastUpdateTimestamp);
      _cache.nextBorrowIndex = cumulatedBorrowInterest.rayMul(_cache.currBorrowIndex).toUint128();
      _reserve.borrowIndex = _cache.nextBorrowIndex;
    }
  }
```

## Impact
It misleads users about debt accumulation, potentially discouraging their lending activities due to an inaccurate perception of financial obligations.

## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L69-L79

## Tool used
Manual Review

## Recommendation
`getNormalizedDebt` function should return `reserve.borrowIndex` when there's no debtShare.