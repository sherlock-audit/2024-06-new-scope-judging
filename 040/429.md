Joyous Rainbow Shark

Medium

# Unclaimable reserve assets will accrue in a pool due to the difference between interest paid on borrows and interest earned on supplies

### Summary

The interest paid on borrows is calculated in a compounding fashion, but the interest earned on supplying assets is calculated in a fixed way. As a result more interest will be repaid by borrowers than is claimable by suppliers. This buildup of balance never gets rebased into the `liquidityIndex`, nor is it claimable with some sort of 'skim' function.

### Root Cause

Any time an action calls `ReserveLogic::updateState()`, [`ReserveLogic::_updateIndexes()` is called](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L91).

In `_updateIndexes()`, the `_cache.nextLiquidityIndex` is a [scaled up](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L226) version of `_cache.currLiquidityIndex` based on the 'linear interest' [calculated in `MathUtils::calculateLinearInterest()`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L225).

[`calculateLinearInterest`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/utils/MathUtils.sol#L34-L42) scales a fixed interest annual interest rate by the amount of time elapsed since the last call:

```javascript
  function calculateLinearInterest(uint256 rate, uint40 lastUpdateTimestamp) internal view returns (uint256) {
    //solium-disable-next-line
    uint256 result = rate * (block.timestamp - uint256(lastUpdateTimestamp));
    unchecked {
      result = result / SECONDS_PER_YEAR;
    }

    return WadRayMath.RAY + result;
  }
```


Similarly, the `_cache.nextBorrowIndex` is a [scaled up](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L236) version of `_cache.currBorrowIndex` based on the 'compound interest' [calculated in `MathUtils::calculateCompoundedInterest()`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L235).

[`calculateCompoundedInterest`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/utils/MathUtils.sol#L58C12-L89) compounds a rate based on the time elapsed since it was last called:

```javascript
  function calculateCompoundedInterest(uint256 rate, uint40 lastUpdateTimestamp, uint256 currentTimestamp) internal pure returns (uint256) {
    //solium-disable-next-line
    uint256 exp = currentTimestamp - uint256(lastUpdateTimestamp);

    if (exp == 0) {
      return WadRayMath.RAY;
    }

    uint256 expMinusOne;
    uint256 expMinusTwo;
    uint256 basePowerTwo;
    uint256 basePowerThree;
    unchecked {
      expMinusOne = exp - 1;
      expMinusTwo = exp > 2 ? exp - 2 : 0;

      basePowerTwo = rate.rayMul(rate) / (SECONDS_PER_YEAR * SECONDS_PER_YEAR);
      basePowerThree = basePowerTwo.rayMul(rate) / SECONDS_PER_YEAR;
    }

    uint256 secondTerm = exp * expMinusOne * basePowerTwo;
    unchecked {
      secondTerm /= 2;
    }
    uint256 thirdTerm = exp * expMinusOne * expMinusTwo * basePowerThree;
    unchecked {
      thirdTerm /= 6;
    }

    return WadRayMath.RAY + (rate * exp) / SECONDS_PER_YEAR + secondTerm + thirdTerm;
  }
```

As a result, more interest is payable on debt than is earned on supplied liquidity. This is a design choice by the protocol, however without a function to 'skim' this extra interest, these tokens will buildup and are locked in the protocol. 

### Internal pre-conditions

1. Pool operates normally with supplies/borrows/repays
2. `updateState()` must NOT be called every second, as this would create a compounding-effect on the 'linear rate' such that the difference in interest paid on debts is equal to the interest earned on supplies.

### External pre-conditions

_No response_

### Attack Path

1. Several users supply tokens to a pool as normal
2. Users borrow against the liquidity
3. Time passes, all borrows are repaid
4. All suppliers withdraw their funds (as part of this operation the treasury also withdraws their fee assets)
5. A pool remains with 0 supplyShares and 0 debtShares, but still has a token balance which is unclaimable by anyone

### Impact

1. Token buildup in contract is unclaimable by anyone
   - The built up token balance can be borrowed and flash loaned, leading to compounding build up of unclaimable liquidity


### PoC

Create a new file in /test/forge/core/pool and paste the below contents. The test shows a simple supply/borrow/warp/repay flow. After the actions are complete, the pool has more `tokenB` than is claimable by the supplier and the treasury. These tokens are now locked in the contract 

```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {console2} from 'forge-std/src/Test.sol';
import {PoolLiquidationTest} from './PoolLiquidationTests.t.sol';

contract AuditUnclaimableBalanceBuildupOnPool is PoolLiquidationTest {

  function test_POC_UnclaimableBalanceBuildupOnPool () public {
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

    vm.warp(block.timestamp + 365 days); // Time passes, interest accrues, treasury shares accrue
    pool.forceUpdateReserves();

    // Alice full repay
    vm.startPrank(alice);
    tokenB.approve(address(pool), type(uint256).max);
    pool.repaySimple(address(tokenB), type(uint256).max, 0);

    (,,, uint256 debtShares) = pool.marketBalances(address(tokenB));
    // All debt has been repaid
    assert(debtShares == 0); 

    // Bob's claim on pool's tokenB
    bytes32 bobPos = keccak256(abi.encodePacked(bob, 'index', uint256(0)));
    uint256 BobsMaxWithdrawAssets = pool.supplyShares(address(tokenB), bobPos) * pool.getReserveData(address(tokenB)).liquidityIndex / 1e27;

    // Treasury claim on pool's tokenB
    uint256 accruedTreasuryAssets = pool.getReserveData(address(tokenB)).accruedToTreasuryShares * pool.getReserveData(address(tokenB)).liquidityIndex / 1e27;

    // Total balance of pool's tokenB
    uint256 poolTokenBBalance = tokenB.balanceOf(address(pool));

    assert(poolTokenBBalance > BobsMaxWithdrawAssets + accruedTreasuryAssets); // There are more tokenB on the pool than all suppliers + treasury claim. 
  }

}
```

### Mitigation

One option is to create a function which claims the latent funds to the treasury, callable by an owner
- Calls `forceUpdateReserves()`
- Calls `executeMintToTreasury()`
- Calculates the latent funds on a pool's reserve (something like `tokenA.balanceOf(pool) - ( totalSupplyShares * liquidityIndex )`)
- Sends these funds to the treasury

Another option would be to occasionally rebase `liquidityIndex` to increase the value of supplyShares so supplies have a claim on these extra funds.

In both cases it may be sensible to leave some dust as a buffer. 