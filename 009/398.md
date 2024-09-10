Clever Ebony Halibut

Medium

# Position Risk Management Functionality Missing in Position Manager and dos in certain conditions

## Summary
Protocol users who manage their positions through the  `PositionManager` are not able to manage risk of their positions, by setting collateral to on and off. Which is a core functionality of every lending protocol. The missing functionality will doss users from withdrawing also in certain conditions.
## Vulnerability Detail
For each collateral resrve the pool tracks whethere the user is using as collateral or not, this is set in the userConfigMap. Any user could set which reserve he is setting as collateral by calling the 

```solidity
function setUserUseReserveAsCollateral(address asset, uint256 index, bool useAsCollateral) external {
    _setUserUseReserveAsCollateral(asset, index, useAsCollateral);
  }
```
The PositionManager.sol which the protocol users, are expected to interact with, doesn't implement the setUserUseReserveAsCollateral(), which first of all leads to the inablity of protocol users to manage risk on their Positions. 
The second impact and the most severe is that Position holders will be dossed, in the protocols if the ltv of one of the reserve token being used, will be set to zero. In such an event, users are required to set the affected collateral to false in order to do operations that lowers the ltv like withdraw to function.

The doss will be done in the function `validateHFandLtv()` which will be called to check the health of a position is maintend after a withdrawal

```solidity
  function validateHFAndLtv(
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.UserConfigurationMap memory userConfig,
    DataTypes.ExecuteWithdrawParams memory params
  ) internal view {
    DataTypes.ReserveData memory reserve = reservesData[params.asset];

    (, bool hasZeroLtvCollateral) = validateHealthFactor(_balances, reservesData, reservesList, userConfig, params.position, params.pool);

@>>    require(!hasZeroLtvCollateral || reserve.configuration.getLtv() == 0, PoolErrorsLib.LTV_VALIDATION_FAILED);
  }
```
In this case, if the user wants to withdraw other reserves that don't have 0 tlv, the transaction will revert.


## Impact
- missing core functions, that NFTPositionManager users are not able to use
- NFTPositionManager are unable to manage to risk at all
- Withdrawal operations in NFTPositionManager will be dossed in certain conditions

## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/Pool.sol#L175C1-L177C4
## Tool used

Manual Review

## Recommendation
Implement the missing functionality in the `NFTPositionManager.sol`, to allow users to manage the risk on their `NFTPosition`