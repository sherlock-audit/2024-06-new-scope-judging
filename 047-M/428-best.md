Joyous Rainbow Shark

Medium

# Pool supply interest compounds randomly based on the frequency of `ReserveLogic::updateState()` calls, resulting in inconsistant and unexpected returns for suppliers

### Summary

`_updateIndexes()` considers the previously earned interest on supplies as principle on the next update, resulting in a random compounding effect on interest earned. Borrowers may be incentivized to call `forceUpdateReserve()` frequently if gas costs are low relative to the upside of compounding their interest. 

### Root Cause

Any time an action calls `ReserveLogic::updateState()`, [`ReserveLogic::_updateIndexes()` is called](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L91). Note this occurs before important state-changing pool actions as well as through public/external functions such as [`Pool::forceUpdateReserve()`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/Pool.sol#L161-L165) and [`Pool::forceUpdateReserves()`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/Pool.sol#L168-L172).

`_updateIndexes()` calculates lienar interest since the last update and uses this to [scale up the `_cache.currLiquidityIndex` and assigns to `_cache.nextLiquidityIndex`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L225-L227):

```javascript
  function _updateIndexes(DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
    ... SKIP!...

    if (_cache.currLiquidityRate != 0) {
@>    uint256 cumulatedLiquidityInterest = MathUtils.calculateLinearInterest(_cache.currLiquidityRate, _cache.reserveLastUpdateTimestamp);
      _cache.nextLiquidityIndex = cumulatedLiquidityInterest.rayMul(_cache.currLiquidityIndex).toUint128();
      _reserve.liquidityIndex = _cache.nextLiquidityIndex;
    }

    ... SKIP!...
  }
```

This creates a compounding-effect on an intended simple interest calculated, based on the frequency it's called. This is due to the interest accumulated on the last refresh being considered principle on the next refresh.

### Internal pre-conditions

1. Pool functions normally with supply/borrow/repay calls

### External pre-conditions

_No response_

### Attack Path

1. Supplier supplies assets
2. Borrower takes debt against supplier's assets and pays interest
3. Indexes are updated at some frequency
4. Supplier earns compound-like interest on their supplied assets 


### Impact

- Suppliers receive compound-interest on their supplied assets instead of the intended fixed/linear interest.

### PoC

Create a new file in /test/forge/core/pool and paste the below contents.

Run command `forge test --mt test_POC_InterestRateIndexCalc -vv` to see the following logs which show the more frequently the update, the more compounding-like the `liquidityIndex` becomes.

```javascript
Ran 3 tests for test/forge/core/pool/RandomCompoundingInterestPOC.t.sol:PoolSupplyRandomCompoundingEffect
[PASS] test_POC_InterestRateIndexCalc_1_singleUpdate() (gas: 740335)
Logs:
  Updates once after a year: liquidityIndex 1.333e27
  Updates once after a year: borrowIndex 1.446891914398940457716504e27

[PASS] test_POC_InterestRateIndexCalc_2_monthlyUpdates() (gas: 1094439)
Logs:
  Updates monthly for a year: liquidityIndex 1.388987876426245531179679454e27
  Updates monthly for a year: borrowIndex 1.447734004896004725108257228e27

[PASS] test_POC_InterestRateIndexCalc_3_dailyUpdates() (gas: 65326938)
Logs:
  Updates every four hours for a year: liquidityIndex 1.395111981380752339971733874e27
  Updates every four hours for a year: borrowIndex 1.447734611520782405003308237e27
```


```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {console2} from 'forge-std/src/Test.sol';
import {PoolLiquidationTest} from './PoolLiquidationTests.t.sol';

contract PoolSupplyRandomCompoundingEffect is PoolLiquidationTest {

  function test_POC_InterestRateIndexCalc_1_singleUpdate() public {
    _mintAndApprove(alice, tokenA, 10_000e18, address(pool)); // alice 10k tokenA
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool)); // bob 2000 tokenB

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 10_000e18, 0); // 10k tokenA alice supply
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0); // 750 tokenB bob supply
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, supplyAmountB, 0); // 750 tokenB borrow for 100% utilization
    vm.stopPrank();

    vm.warp(block.timestamp + 365 days);
    
    pool.forceUpdateReserves(); // updates indicies

    console2.log('Updates once after a year: liquidityIndex %e', pool.getReserveData(address(tokenB)).liquidityIndex);
    console2.log('Updates once after a year: borrowIndex %e', pool.getReserveData(address(tokenB)).borrowIndex);
  }

  function test_POC_InterestRateIndexCalc_2_monthlyUpdates() public {
    _mintAndApprove(alice, tokenA, 10_000e18, address(pool)); // alice 10k tokenA
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool)); // bob 2000 tokenB

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 10_000e18, 0); // 10k tokenA alice supply
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0); // 750 tokenB bob supply
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, supplyAmountB, 0); // 750 tokenB borrow for 100% utilization
    vm.stopPrank();

    for(uint256 i = 0; i < 12; i++) {
        vm.warp(block.timestamp + (30 * 24 * 60 * 60));
        pool.forceUpdateReserves(); // updates indicies
    }
    vm.warp(block.timestamp + (5 * 24 * 60 * 60)); // Final 5 days
    pool.forceUpdateReserves(); // updates indicies

    console2.log('Updates monthly for a year: liquidityIndex %e', pool.getReserveData(address(tokenB)).liquidityIndex);
    console2.log('Updates monthly for a year: borrowIndex %e', pool.getReserveData(address(tokenB)).borrowIndex);

  }

  function test_POC_InterestRateIndexCalc_3_dailyUpdates() public {
    _mintAndApprove(alice, tokenA, 10_000e18, address(pool)); // alice 10k tokenA
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool)); // bob 2000 tokenB

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 10_000e18, 0); // 10k tokenA alice supply
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0); // 750 tokenB bob supply
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, supplyAmountB, 0); // 750 tokenB borrow for 100% utilization
    vm.stopPrank();

    for(uint256 i = 0; i < (6 * 365); i++) {
        vm.warp(block.timestamp + (4 * 60 * 60));
        pool.forceUpdateReserves(); // updates indicies
    }

    console2.log('Updates every four hours for a year: liquidityIndex %e', pool.getReserveData(address(tokenB)).liquidityIndex);
    console2.log('Updates every four hours for a year: borrowIndex %e', pool.getReserveData(address(tokenB)).borrowIndex);
  }
}
```

### Mitigation

- Track the principle and the interest separately. Accrue simple interest on the principle only.