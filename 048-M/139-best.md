Decent Walnut Squirrel

High

# Protocol does not consider rounding direction.

## Summary
The `rayDiv` function which is used for calculation of share from amount can use rounding up or rounding down, occasionally. An attacker can steal more funds than normal.

## Vulnerability Detail
`PositionBalanceConfiguration.sol#depositCollateral()` function is as follows.
```solidity
  function depositCollateral(
    DataTypes.PositionBalance storage self,
    DataTypes.ReserveSupplies storage totalSupply,
    uint256 amount,
    uint128 index
  ) internal returns (bool isFirst, uint256 sharesMinted) {
@>  sharesMinted = amount.rayDiv(index);
    require(sharesMinted != 0, PoolErrorsLib.INVALID_MINT_AMOUNT);
    isFirst = self.supplyShares == 0;
    self.lastSupplyLiquidtyIndex = index;
    self.supplyShares += sharesMinted;
    totalSupply.supplyShares += sharesMinted;
  }
```
And `PositionBalanceConfiguration.sol#withdrawCollateral()` function is as follows.
```solidity
  function withdrawCollateral(
    DataTypes.PositionBalance storage self,
    DataTypes.ReserveSupplies storage supply,
    uint256 amount,
    uint128 index
  ) internal returns (uint256 sharesBurnt) {
@>  sharesBurnt = amount.rayDiv(index);
    require(sharesBurnt != 0, PoolErrorsLib.INVALID_BURN_AMOUNT);
    self.lastSupplyLiquidtyIndex = index;
    self.supplyShares -= sharesBurnt;
    supply.supplyShares -= sharesBurnt;
  }
```
`WadRayMath.sol#rayDiv()` function which is called above is as follows.
```solidity
  function rayDiv(uint256 a, uint256 b) internal pure returns (uint256 c) {
    // to avoid overflow, a <= (type(uint256).max - halfB) / RAY
    assembly {
      if or(iszero(b), iszero(iszero(gt(a, div(sub(not(0), div(b, 2)), RAY))))) { revert(0, 0) }

      c := div(add(mul(a, RAY), div(b, 2)), b)
    }
  }
```
As we can see above, `rayDiv` can use rounding up or down, occasionally.   
So an attacker can steal more funds than normal.    
Especially, this problem is more serious when asset has low decimals.   
This problem exists in borrwing and repaying.   

Let's see following example.   
We say `index = 1.1e27`.   
A user deposits 3 wei, then he gets share: `(3 * 1e27 + 0.55e27) / 1.1e27 = 3`.   
Then the user deposits 3 wei, again. Then, his total supply share: 6. (But total amount becomes `(6 * 1.1e27 + 0.5e27) / 1e27 = 7` - See `rayMul()`)   
And he withdraws 6, then burnt share: `(6 * 1e27 + 0.55e27) / 1.1e27 = 5`.   
So 1 wei of share remains to the user.   
Therefore, user's incentive percent is `1 * 100 / 6 = 16.6%`.

## Impact
An attacker can steal more funds than normal for asset with low decimals by using protocol's weakness of rounding process.

## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L44

## Tool used

Manual Review

## Recommendation
Use rounding down when deposit and repay, rounding up when withdraw and borrow.