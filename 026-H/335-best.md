Soft Rose Cod

High

# Liquidations do not target the lowest LTV tokens leading to a position losing health

### Summary

When users liquidate a position, they are able to choose one debt asset to pay off and one asset of the users to seize for a discount. Since there are no restrictions on which collateral asset a user can seize, they can choose a high LTV asset that will often lower a positions health.

### Root Cause

The liquidation mechanism does not prioritize the lowest LTV assets when liquidating. Liquidators are given the ability to choose the safer higher LTV assets as collateral to seize which results in the positions health dropping.

Example:
Consider a position with collateral asset A, C and debt asset B and a 0% liquidation bonus. A is a safe asset with a liquidation LTV of 90% while C is a risky asset with a liquidation LTV of 50%. If the position has 1000 USDC of A and 1000 USDC of C then their total borrowing power is (1000*.90 + 1000*.5) = $1500 total borrowing power. If the position has a debt value greater than $1500 they will be liquidated. Consider the case where they have $1600 in debt, A positions health is calculated as:

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/GenericLogic.sol#L141C1-L143

```HealthFactor = (average liquidation LTV) * (total collateral value) / (total debt value)```
```HealthFactor = (1000*.90+1000*.5)/(1600) = 1500/1600 = .875```
Since ```HealthFactor < 1``` then the position is liquidatable. If the liquidator were to seize asset A to pay off $1000 of debt then the new ```HealthFactor``` is:
```HealthFactor = (1000*.5)/(600) = 5/6 = .83333```
Even with a 0% liquidation bonus, the positions ```HealthFactor``` decreased from .875 to .8333 after liquidation and the position is even more at risk of creating bad debt.


### Internal pre-conditions

1. User has to have a position with multiple collateral assets of varying LTV's and is borrowing an asset
2. the user has to be liquidated
3. the liquidator has to seize the higher LTV asset

### External pre-conditions

1. user collateral has to drop enough in price to be liquidatable
2. user has to get liquidated

### Attack Path

1. User has a position with multiple collateral assets of varying LTV's and is borrowing an asset
2. The positions health factor drops below 1
3. The liquidator liquidates the position and chooses the higher LTV (safer) asset
4. Liquidator

### Impact

Two main impacts:
1. The protocol is at greater risk of bad debt and in some cases is guaranteed bad debt.
2. The liquidator can bypass partial liquidation checks (.95 < HF < 1) by liquidating once and causing the HF to decrease below .95. The liquidator can then liquidate again to seize the rest of their collateral.

### PoC

Steps:
1. set the ```PoolSetup.sol#_basicPoolInitParams``` function to the following. This is to change the LTV's/liquidation thresholds of the assets.
```solidity
  function _basicPoolInitParams() internal view returns (DataTypes.InitPoolParams memory p) {

    address[] memory assets = new address[](4);
    assets[0] = address(tokenA);
    assets[1] = address(tokenB);
    assets[2] = address(tokenC);
    assets[3] = address(wethToken);

    address[] memory rateStrategyAddresses = new address[](4);
    rateStrategyAddresses[0] = address(irStrategy);
    rateStrategyAddresses[1] = address(irStrategy);
    rateStrategyAddresses[2] = address(irStrategy);
    rateStrategyAddresses[3] = address(irStrategy);

    address[] memory sources = new address[](4);
    sources[0] = address(oracleA);
    sources[1] = address(oracleB);
    sources[2] = address(oracleC);
    sources[3] = address(oracleD);

    DataTypes.InitReserveConfig memory config = _basicConfig();

    DataTypes.InitReserveConfig[] memory configurationLocal = new DataTypes.InitReserveConfig[](4);
    DataTypes.InitReserveConfig memory configA = DataTypes.InitReserveConfig({
      ltv: 9000,
      liquidationThreshold: 9500,
      liquidationBonus: 10_500,
      decimals: 18,
      frozen: false,
      borrowable: true,
      borrowCap: 0,
      supplyCap: 0
    });

    DataTypes.InitReserveConfig memory configC = DataTypes.InitReserveConfig({
      ltv: 5000,
      liquidationThreshold: 8000,
      liquidationBonus: 10_500,
      decimals: 18,
      frozen: false,
      borrowable: true,
      borrowCap: 0,
      supplyCap: 0
    });
    configurationLocal[0] = configA;
    configurationLocal[1] = config;
    configurationLocal[2] = configC;
    configurationLocal[3] = config;

    address[] memory admins = new address[](1);
    admins[0] = address(this);

    p = DataTypes.InitPoolParams({
      proxyAdmin: address(this),
      revokeProxy: false,
      admins: admins,
      emergencyAdmins: new address[](0),
      riskAdmins: new address[](0),
      hook: address(0),
      assets: assets,
      rateStrategyAddresses: rateStrategyAddresses,
      sources: sources,
      configurations: configurationLocal
    });
  }
```
2. place the following test inside ```NFTPositionManagerTest.t.sol```
```solidity
  function test_LiquidateHealthyAssetBug() public {
    uint256 mintAmount = 100 ether;
    uint256 supplyAmount = 1 ether;
    uint256 tokenId = 1;

    //approvals
    _mintAndApprove(alice, tokenA, 1000 ether, address(pool)); // alice 1000 tokenA
    _mintAndApprove(alice, tokenB, 1000 ether, address(pool)); // alice 1000 tokenB
    _mintAndApprove(alice, tokenC, 1000 ether, address(pool)); // alice 1000 tokenC
    _mintAndApprove(bob, tokenB, 5000 ether, address(pool)); // bob 2000 tokenB


    DataTypes.ExtraData memory data = DataTypes.ExtraData(bytes(''), bytes(''));
    INFTPositionManager.AssetOperationParams memory aliceSupplyA =
      INFTPositionManager.AssetOperationParams(address(tokenA), alice, 1000 ether, tokenId, data);
    INFTPositionManager.AssetOperationParams memory aliceSupplyC =
      INFTPositionManager.AssetOperationParams(address(tokenC), alice, 1000 ether, tokenId, data);
    INFTPositionManager.AssetOperationParams memory aliceBorrowB =
      INFTPositionManager.AssetOperationParams(address(tokenB), alice, 1390 ether, tokenId, data);
    INFTPositionManager.AssetOperationParams memory bobSupplyB =
      INFTPositionManager.AssetOperationParams(address(tokenB), bob, 2000 ether, 2, data);
    oracleA.updateAnswer(1e8);
    oracleB.updateAnswer(1e8);
    oracleC.updateAnswer(1e8);

    vm.startPrank(alice);
    tokenA.approve(address(nftPositionManager), 100000 ether);
    tokenB.approve(address(nftPositionManager), 100000 ether);
    tokenC.approve(address(nftPositionManager), 100000 ether);
    nftPositionManager.mint(address(pool));
    nftPositionManager.supply(aliceSupplyA);
    nftPositionManager.supply(aliceSupplyC);
    vm.stopPrank();

    vm.startPrank(bob);
    tokenA.approve(address(pool), 100000 ether);
    tokenB.approve(address(pool), 100000 ether);
    tokenC.approve(address(pool), 100000 ether);
    tokenA.approve(address(nftPositionManager), 100000 ether);
    tokenB.approve(address(nftPositionManager), 100000 ether);
    tokenC.approve(address(nftPositionManager), 100000 ether);
    nftPositionManager.mint(address(pool));
    nftPositionManager.supply(bobSupplyB);
    vm.stopPrank();

    vm.startPrank(alice);
    nftPositionManager.borrow(aliceBorrowB);
    vm.stopPrank();

    oracleC.updateAnswer(5000e4);
    (,,,,uint256 currentLTV,uint256 healthFactorBefore) = pool.getUserAccountData(address(nftPositionManager), 1);
    bytes32 pos = keccak256(abi.encodePacked(nftPositionManager, 'index', uint256(1)));
    console.log("\n========================\n");
    console.log("alice HF before liquidation: %e", healthFactorBefore);
    console.log("alice LTV before liquidation: %e\n", currentLTV);

    console.log("Liquidating...");
    vm.startPrank(bob);
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, 10000e18);

    (,,,,uint256 afterLTV,uint256 healthFactorAfter) = pool.getUserAccountData(address(nftPositionManager), 1);
    console.log("\nalice HF after liquidation: %e", healthFactorAfter);
    console.log("alice LTV after liquidation: %e", afterLTV);
    vm.stopPrank();
  }
```

#### Output
  alice HF before liquidation: 9.71223021582733813e17
  alice LTV before liquidation: 7.666e3

  Liquidating...

  alice HF after liquidation: 9.44913884892086331e17
  alice LTV after liquidation: 6.403e3

The PoC shows a decrease in HF from .971 to .944 along with a decrease in LTV.

### Mitigation

Liquidation should prioritize the lowest LTV/riskiest assets first.