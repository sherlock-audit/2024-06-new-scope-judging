Magnificent Scarlet Dove

High

# Incorrect Validation logic is used in validate supply function

## Summary
Incorrect Validation logic is used in validate supply function
## Vulnerability Detail
Following is validate supply function 
```solidity
 function validateSupply(
    DataTypes.ReserveCache memory cache,
    DataTypes.ReserveData storage reserve,
    DataTypes.ExecuteSupplyParams memory params,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal view {
    require(params.amount != 0, PoolErrorsLib.INVALID_AMOUNT);

    (bool isFrozen,) = cache.reserveConfiguration.getFlags();
    require(!isFrozen, PoolErrorsLib.RESERVE_FROZEN);

    uint256 supplyCap = cache.reserveConfiguration.getSupplyCap();

    require(
      supplyCap == 0
        || ((totalSupplies.supplyShares + uint256(reserve.accruedToTreasuryShares)).rayMul(cache.nextLiquidityIndex) + params.amount)
          <= supplyCap * (10 ** cache.reserveConfiguration.getDecimals()),
      PoolErrorsLib.SUPPLY_CAP_EXCEEDED
    );
  }
```
Issue is in the following lines 
```solidity
require(
      supplyCap == 0
        || ((totalSupplies.supplyShares + uint256(reserve.accruedToTreasuryShares)).rayMul(cache.nextLiquidityIndex) + params.amount)
          <= supplyCap * (10 ** cache.reserveConfiguration.getDecimals()),
      PoolErrorsLib.SUPPLY_CAP_EXCEEDED
    );
```
As can be seen that totalSupplies.supplyShares  represent shares whereas uint256(reserve.accruedToTreasuryShares)).rayMul(cache.nextLiquidityIndex)  represent total assets accrued to treasury
and params.amount reperesents the amount of assets user wants to supply so all of the above three variables are not in same units one is in shares and other two are actual assets which leads to inconsistency in code logic.

As can be seen in validate borrow function 
```solidity
function validateBorrow(
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.ValidateBorrowParams memory params
  ) internal view {
    require(params.amount != 0, PoolErrorsLib.INVALID_AMOUNT);

    ValidateBorrowLocalVars memory vars;

    (vars.isFrozen, vars.borrowingEnabled) = params.cache.reserveConfiguration.getFlags();

    require(!vars.isFrozen, PoolErrorsLib.RESERVE_FROZEN);
    require(vars.borrowingEnabled, PoolErrorsLib.BORROWING_NOT_ENABLED);

    vars.reserveDecimals = params.cache.reserveConfiguration.getDecimals();
    vars.borrowCap = params.cache.reserveConfiguration.getBorrowCap();
    unchecked {
      vars.assetUnit = 10 ** vars.reserveDecimals;
    }

    if (vars.borrowCap != 0) {
      vars.totalSupplyVariableDebt = params.cache.currDebtShares.rayMul(params.cache.nextBorrowIndex);

      vars.totalDebt = vars.totalSupplyVariableDebt + params.amount;

      unchecked {
        require(vars.totalDebt <= vars.borrowCap * vars.assetUnit, PoolErrorsLib.BORROW_CAP_EXCEEDED);
      }
    }.........
rest of the code 
  }
```
We can clearly see that vars.totalDebt = vars.totalSupplyVariableDebt + params.amount;
 and vars.totalSupplyVariableDebt = params.cache.currDebtShares.rayMul(params.cache.nextBorrowIndex); represents the actual debt accrued not debt shares.

Similiar logic is not followed while validating the supply logic therefore incorrect check in validate supply function
## Impact
Incorrect check will allow to supply more than the supply cap.

## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ValidationLogic.sol#L85
## Tool used

Manual Review

## Recommendation
Change the check as follows
```solidity
 require(
      supplyCap == 0
        || ((totalSupplies.supplyShares.rayMul(cache.nextLiquidityIndex) + uint256(reserve.accruedToTreasuryShares)).rayMul(cache.nextLiquidityIndex) + params.amount)
          <= supplyCap * (10 ** cache.reserveConfiguration.getDecimals()),
      PoolErrorsLib.SUPPLY_CAP_EXCEEDED
    );
```