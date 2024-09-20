Polite Garnet Swift

Medium

# `_convertToShares` function adds `feeShares` to the total supply before converting assets to shares

### Summary

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVaultGetters.sol#L149

in function  `_convertToShares`

The issue relates to how performance fees are accounted for when converting assets to shares.

Here's the relevant part of the  function:

```solidity
function _convertToShares(uint256 assets, MathUpgradeable.Rounding rounding) internal view override returns (uint256) {
  (uint256 feeShares, uint256 newTotalAssets) = _accruedFeeShares();
  return _convertToSharesWithTotals(assets, totalSupply() + feeShares, newTotalAssets, rounding);
}
```

The potential bug is in how `feeShares` are added to the total supply when calculating the conversion. This approach can lead to an inflation attack, which is actually mentioned in the QA section of the documentation:

> We also suspect a possible inflation attack with the vaults as described with the OpenZeppelin's doc on ERC4626: https://docs.openzeppelin.com/contracts/4.x/erc4626#inflation-attack if this can be proven then it'll be a valid bug

The issue arises because the function adds `feeShares` to the total supply before converting assets to shares. This can be exploited by an attacker in the following way:

1. The attacker deposits a small amount of assets into the vault when it's empty or has very low total assets.
2. This causes `feeShares` to be minted, inflating the total supply.
3. When the next user tries to deposit, they will receive fewer shares than they should because the total supply has been artificially inflated.

This bug can have a significant impact on the project as it can lead to unfair distribution of shares and potentially cause users to receive fewer shares than they should for their deposited assets.

To fix this issue, the vault should consider implementing one of the mitigations suggested by OpenZeppelin, such as using a minimum deposit amount or implementing a virtual offset. Alternatively, the fee accrual mechanism could be redesigned to avoid this vulnerability


### Potential for triggering:

particularly when the vault has very low total assets.


Step 1: Initial state
- Assume the vault is empty (totalAssets = 0, totalSupply = 0)
- The underlying asset has 18 decimals

Step 2: Attacker's action
- Attacker deposits 1 wei of the asset (0.000000000000000001 tokens)
- This updates `lastTotalAssets` to 1

Step 3: Price increase
- Let's say the price of the underlying asset increases significantly
- Now, totalAssets() returns 1000 (representing 0.000000000000001 tokens)

Step 4: Fee accrual
- When `_accruedFeeShares` is called, it calculates:
  totalInterest = 1000 - 1 = 999
  feeAssets = 999 * fee (let's say fee is 20%) = 199
  feeShares are minted based on this 199 feeAssets

Step 5: Victim's deposit
- A victim tries to deposit 1000 assets (0.000000000000001 tokens)
- `_convertToShares` is called, which adds the feeShares to totalSupply
- The victim receives significantly fewer shares than they should, due to the inflated totalSupply

### Impact:
1. Early depositors can manipulate the share price, causing later depositors to receive fewer shares than they should.
2. This can lead to a loss of funds for later depositors, as they're effectively paying for an inflated fee that doesn't represent real growth of the vault's assets.

While the virtual shares/assets provide some protection, they may not be sufficient in extreme cases or with assets that can experience significant price volatility.

### Conclusion :

The vulnerability could be triggered, although the specific conditions required (very low initial deposits followed by significant price increases) might not be common. However, given the potential for financial loss and the importance of fairness in DeFi protocols, this should be considered a serious issue that needs addressing.