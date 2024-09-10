Rapid Onyx Meerkat

Medium

# new reserve configuration can lead to user's postion being liquidated

### Summary

When setting a new configuration for a specific reserve using a configurator, insufficient validation may lead to undesirable outcomes. This could result in a user's position being unexpectedly liquidated due to changes in the reserve’s configuration

### Root Cause

[PoolLogic.sol::setReserveConfiguration](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/PoolLogic.sol#L139-L177)  from `setReserveConfiguration` we can see only `config.getLtv() <= config.getLiquidationThreshold()` is validated
```solidity
  function setReserveConfiguration(
    mapping(address => DataTypes.ReserveData) storage _reserves,
    address asset,
    address rateStrategyAddress,
    address source,
    DataTypes.ReserveConfigurationMap memory config
  ) public {
    require(asset != address(0), PoolErrorsLib.ZERO_ADDRESS_NOT_VALID);
    _reserves[asset].configuration = config;

    // set if values are non-0
    if (rateStrategyAddress != address(0)) _reserves[asset].interestRateStrategyAddress = rateStrategyAddress;
    if (source != address(0)) _reserves[asset].oracle = source;

    require(config.getDecimals() >= 6, 'not enough decimals');

    // validation of the parameters: the LTV can
    // only be lower or equal than the liquidation threshold
    // (otherwise a loan against the asset would cause instantaneous liquidation)
    require(config.getLtv() <= config.getLiquidationThreshold(), PoolErrorsLib.INVALID_RESERVE_PARAMS);
```

### Internal pre-conditions

1.configurator want to set `ltv` and `LiquidationThreshold` at the same time
2.import `{ReserveConfiguration}` from file `ReserveConfiguration.sol`

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

User's position being liquidated, which lead to user lost of funds

### PoC

add test to file `NFTPositionManagerTest.t.sol`
```solidity
  function testLiquidationThresholdDecrease() public {
    vm.warp(1641070800);

    //candy supply tokenB for borrow.
    uint256 mintAmount = 100 ether;
    DataTypes.ExtraData memory data = DataTypes.ExtraData(bytes(''), bytes(''));
    INFTPositionManager.AssetOperationParams memory params =
      INFTPositionManager.AssetOperationParams(address(tokenB), candy, 20e18, 1, data);
    _mintAndApprove(candy, tokenB, mintAmount, address(nftPositionManager));

    vm.startPrank(candy);
    nftPositionManager.mint(address(pool));
    nftPositionManager.supply(params);
    vm.stopPrank();

    //alice supply tokenA.
    params = INFTPositionManager.AssetOperationParams(address(tokenA), alice, 20e18, 2, data);
    _mintAndApprove(alice, tokenA, mintAmount, address(nftPositionManager));

    vm.startPrank(alice);
    nftPositionManager.mint(address(pool));
    nftPositionManager.supply(params);

    //alice borrow
    params = INFTPositionManager.AssetOperationParams(address(tokenB), alice, 7.5e18, 2, data);
    nftPositionManager.borrow(params);

    //config decrease threshold.
    vm.stopPrank();
    DataTypes.ReserveConfigurationMap memory self = pool.getConfiguration(address(tokenA));
    self.setLtv(7000);
    self.setLiquidationThreshold(7400);
    vm.prank(address(poolFactory.configurator()));
    pool.setReserveConfiguration(address(tokenA),address(0),address(0),self);

    (,,,,,uint256 healthFactor) = pool.getUserAccountData(address(nftPositionManager),2);
    assert(healthFactor < 1e18);
  }

```

### Mitigation

add new validation ensure new `LiquidationThreshold` is greater than previos `ltv`
```diff
@@ -144,6 +143,7 @@ library PoolLogic {
     DataTypes.ReserveConfigurationMap memory config
   ) public {
     require(asset != address(0), PoolErrorsLib.ZERO_ADDRESS_NOT_VALID);
+    uint256 oldLtv = _reserves[asset].configuration.getLtv();
     _reserves[asset].configuration = config;
 
     // set if values are non-0
@@ -156,6 +156,7 @@ library PoolLogic {
     // only be lower or equal than the liquidation threshold
     // (otherwise a loan against the asset would cause instantaneous liquidation)
     require(config.getLtv() <= config.getLiquidationThreshold(), PoolErrorsLib.INVALID_RESERVE_PARAMS);
+    require(config.getLiquidationThreshold() > oldLtv,PoolErrorsLib.INVALID_RESERVE_PARAMS);
```