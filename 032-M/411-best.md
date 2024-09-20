Smooth Carbon Narwhal

Medium

# Withdrawals can be reverted in the curated vault

## Summary
Users can `deposit` `assets` into selected `pools` via a `curated vault`. 
By doing so, they may `withdraw` more `assets`, as `interest` accumulates in the `pools` over time. 
However, `withdrawals` may sometimes fail, making it difficult to resolve the issue.
## Vulnerability Detail
When users request a `withdrawal` through the `curated vault`, the `vault` pulls `assets` from the selected `pools` one by one. 
```solidity
function _withdrawPool(uint256 withdrawAmount) internal {
  for (uint256 i; i < withdrawQueue.length; ++i) {
    IPool pool = withdrawQueue[i];
    (uint256 supplyAssets,) = _accruedSupplyBalance(pool);
150:    uint256 toWithdraw =
      UtilsLib.min(_withdrawable(pool, pool.totalAssets(asset()), pool.totalDebt(asset()), supplyAssets), withdrawAmount);
  }

  if (withdrawAmount != 0) revert CuratedErrorsLib.NotEnoughLiquidity();
}
```
On `line 150`, the `withdrawable` amount from each `pool` is calculated. 
In the `_withdrawable` function, there is a potential for `underflow` if `totalSupplyAssets` is less than `totalBorrowAssets`, preventing users from `withdrawing` their `assets`.
```solidity
function _withdrawable(
  IPool pool,
  uint256 totalSupplyAssets,
  uint256 totalBorrowAssets,
  uint256 supplyAssets
) internal view returns (uint256) {
  uint256 availableLiquidity = UtilsLib.min(totalSupplyAssets - totalBorrowAssets, IERC20(asset()).balanceOf(address(pool)));
  return UtilsLib.min(supplyAssets, availableLiquidity);
}
```

It’s important to note that the `pool` doesn’t guarantee that `totalSupplyAssets` will always exceed `totalBorrowAssets`. 
Some users interact directly with these `pools` without going through the `curated vault`. 
If all supplied `assets` are `borrowed` and the `borrow index` exceeds the `liquidity index`, or if depositors `withdraw` as much as possible while `borrowers` maintain their positions, `totalSupplyAssets` can drop below `totalBorrowAssets`.

To address this issue, the `vault` can remove the `pool` from the `withdrawQueue`. 
However, this action might cause users to lose their funds, as `assets` supplied to the `pool` would no longer be factored in.
## Impact
This is a `DoS`.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L150
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultGetters.sol#L126-L127
## Tool used

Manual Review

## Recommendation
```solidity
function _withdrawable(
  IPool pool,
  uint256 totalSupplyAssets,
  uint256 totalBorrowAssets,
  uint256 supplyAssets
) internal view returns (uint256) {
-  uint256 availableLiquidity = UtilsLib.min(totalSupplyAssets - totalBorrowAssets, IERC20(asset()).balanceOf(address(pool)));
+  uint256 availableLiquidity = totalSupplyAssets < totalBorrowAssets ? 0 : UtilsLib.min(totalSupplyAssets - totalBorrowAssets, IERC20(asset()).balanceOf(address(pool)));

  return UtilsLib.min(supplyAssets, availableLiquidity);
}
```