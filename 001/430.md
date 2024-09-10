Joyous Rainbow Shark

Medium

# Supply interest is earned on `accruedToTreasuryShares` resulting in higher than expected treasury fees and under rare circumstances DOSed pool withdrawals

### Summary

Fees on debt interest are calculated in 'assets', but not claimed immediately and stored as shares. When they're claimed, they're treated as regular supplyShares and converted back to assets based on the `liquidityIndex` at the time they're claimed. This results in the realized fees being higher than expected, and under an extreme scenario may not leave enough liquidity for regular pool suppliers to withdraw their funds.

### Root Cause

Each time a pool's reserve state is updated, [`ReserveLogic::_accrueToTreasury()` is called](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L92).
This function incriments `_reserve.accruedToTreasuryShares` by the shares equiavalent of the assets taken as a fee. Note `vars.amountToMint` is in assets, and `_reserve.accruedToTreasuryShares` is stored in shares as the 'fee assets' are not always immediately sent to the treasury.

```javascript
  function _accrueToTreasury(uint256 reserveFactor, DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
    ... SKIP!...

    vars.amountToMint = vars.totalDebtAccrued.percentMul(reserveFactor); // Assets

@>  if (vars.amountToMint != 0) _reserve.accruedToTreasuryShares += vars.amountToMint.rayDiv(_cache.nextLiquidityIndex).toUint128(); // Shares
  }
```

When any pool withdrawal is executed, [`PoolLogic::executeMintToTreasury()` is called](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/PoolSetters.sol#L84). The stored shares are converted back to assets based on the current `liquidityIndex`:

```javascript
  function executeMintToTreasury(
    DataTypes.ReserveSupplies storage totalSupply,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    address treasury,
    address asset
  ) external {
    DataTypes.ReserveData storage reserve = reservesData[asset];

    uint256 accruedToTreasuryShares = reserve.accruedToTreasuryShares;

    if (accruedToTreasuryShares != 0) {
      reserve.accruedToTreasuryShares = 0;
      uint256 normalizedIncome = reserve.getNormalizedIncome();
      uint256 amountToMint = accruedToTreasuryShares.rayMul(normalizedIncome); // Assets, scaled up by current liquidityIndex

      IERC20(asset).safeTransfer(treasury, amountToMint);
      totalSupply.supplyShares -= accruedToTreasuryShares;

      emit PoolEventsLib.MintedToTreasury(asset, amountToMint);
    }
  }
```

As a result, the actual fee taken by the treasury exceeds the original x% intended by `reserveFactor` at the time the debt was repaid.

In addition, the accumulation of interest on these `accruedToTreasuryShares` can lead to pool withdrawals being DOSed under circumstances where the `_updateIndexes()` is called close to every second to create a compounding effect. This compounding effect on brings the `liquidityIndex` closer to the `borrowIndex`. This results in the interest earned on `accruedToTreasuryShares` causing the pool to run out of liquidity when suppliers withdraw.

### Internal pre-conditions

Impact 1 
1. A pool has a non-zero `reserveFactor`
2. Pool operates normally with supplies/borrows/repays

Impact 2
1. A pool has a non-zero `reserveFactor`
2. Pool operates normally with supplies/borrows/repays
3. `_updateIndexes()` is called each second (or close to)

### External pre-conditions

_No response_

### Attack Path

Impact 2
1. User1 supplies to a pool
2. User2 borrows from the same pool
3. As time elapses, `_updateIndexes()` is called close to every second, bringing `liquidityIndex` closer to `borrowIndex`. Note this is callable from an external function `Pool::forceUpdateReserves()`
4. User2 repays their borrow including interest
5. Repeat step 3 just for a few seconds
6. User1 attempts to withdraw their balance but due to the accrued interest on `accruedToTreasuryShares`, the pool runs out of liquidity DOSing the withdrawal.

### Impact

1. Generally speaking, in all pools the treasury will end up taking a larger fee than what was set in `reserveFactor`. That is, if `reserveFactor` is 1e3 (10%) and 1e18 interest is earned, the protocol will eventually claim more than 10% * 1e18 assets.
2. Under a specific scenario where `_updateIndexes()` is called every second, there will not be enough liquidity for suppliers to withdraw because the treasury earning supply interest on their `accruedToTreasuryShares` is not accounted for.

### PoC

The below coded POC implements the 'Attack Path' described above.

First, add this line to the `CorePoolTests::_setUpCorePool()` function to create a scenario with a non-zero reserve factor:

```diff
  function _setUpCorePool() internal {
    poolImplementation = new Pool();

    poolFactory = new PoolFactory(address(poolImplementation));
+   poolFactory.setReserveFactor(1e3); // 10%
    configurator = new PoolConfigurator(address(poolFactory));

    ... SKIP!...
  }
```

Then, create a new file on /test/forge/core/pool and paste the below contents.

```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {console2} from 'forge-std/src/Test.sol';
import {PoolLiquidationTest} from './PoolLiquidationTests.t.sol';

contract AuditHighSupplyRateDosWithdrawals is PoolLiquidationTest {

function test_POC_DosedWithdrawalsDueToTreasurySharesAccruing() public {
    uint256 aliceMintAmount = 10_000e18;
    uint256 bobMintAmount = 10_000e18;
    uint256 supplyAmount = 1000e18;
    uint256 borrowAmount = 1000e18;

    _mintAndApprove(alice, tokenA, aliceMintAmount, address(pool));         // alice collateral
    _mintAndApprove(bob, tokenB, bobMintAmount, address(pool));             // bob supply
    _mintAndApprove(alice, tokenB, aliceMintAmount, address(pool));         // alice needs some funds to pay interest

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmount, 0); 
    vm.stopPrank();

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, aliceMintAmount, 0);  // alice collateral
    pool.borrowSimple(address(tokenB), alice, borrowAmount, 0);     // 100% utilization
    vm.stopPrank();

    for(uint256 i = 0; i < (12 * 60 * 60); i++) { // Each second `_updateIndexes()` is called via external function `forceUpdateReserves()`
        vm.warp(block.timestamp + 1);
        pool.forceUpdateReserves();
    }

    // Alice full repay
    vm.startPrank(alice);
    tokenB.approve(address(pool), type(uint256).max);
    pool.repaySimple(address(tokenB), type(uint256).max, 0);
    uint256 post_aliceTokenBBalance = tokenB.balanceOf(alice);
    uint256 interestRepaidByAlice = aliceMintAmount - post_aliceTokenBBalance;

    for(uint256 i = 0; i < (60); i++) { // warp after for treasury to accrue interest on their 'fee shares' 
        vm.warp(block.timestamp + 1);
        pool.forceUpdateReserves();
    }

    // Check debt has been repaid
    (, , , uint256 debtShares) = pool.marketBalances(address(tokenB));
    assert(debtShares == 0); // All debt has been repaid

    // Treasury assets to claim
    uint256 treasuryAssetsToClaim = pool.getReserveData(address(tokenB)).accruedToTreasuryShares * pool.getReserveData(address(tokenB)).liquidityIndex / 1e27;

    // Bob's assets to claim
    bytes32 bobPos = keccak256(abi.encodePacked(bob, 'index', uint256(0)));
    uint256 bobsAssets = pool.supplyShares(address(tokenB), bobPos) * pool.getReserveData(address(tokenB)).liquidityIndex / 1e27;

    // Impact 1: the interest claimable by the treasury is greater than 10% of the interest repaid
    assert(treasuryAssetsToClaim > pool.factory().reserveFactor() * interestRepaidByAlice / 1e4);

    // Impact 2: Bob & the treasury's claim on the assets is greater than available assets, despite no outstanding debt. 
    // This assert demonstrates that bob's withdrawal would be DOSed as withdrawal calls include a transfer of treasury assets.
    // The withdrawal DOS cannot be shown due to the call reverting due to the 'share underflow' issue described in another report
    uint256 poolLiquidity = tokenB.balanceOf(address(pool));
    assert(bobsAssets + treasuryAssetsToClaim > poolLiquidity); 
  }
}
```

### Mitigation

Three possible solutions:
1. Immediately send the 'fee assets' to treasury rather than accruing them over time
2. Store the 'fee assets' in assets instead of shares. This will correctly capture the amount of fee that is intended by `reserveFactor`. For example if a fee is 10%, the protocol will take exactly 10% of the interest earned on debt.
3. Account for the creation of new `supplyShares` by diluting the `liquidityIndex` upon creating these shares. This solution will allow the conversion back to assets in `executeMintToTreasury()` to remain unchanged.
   - Note I acknowledge that the [calculation of `liquidityRate`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/periphery/ir/DefaultReserveInterestRateStrategy.sol#L126-L128) does account for `reserveFactor`, however given this is out of scope I did not focus on it. Regardless, it does not scale the rate down enough to account for the interest the treasury will earn on these shares. 