Joyous Rainbow Shark

Medium

# Curated Vault allocators cannot `reallocate()` a pool to zero due to attempting to withdraw 0 tokens from the underlying pool

### Summary

The `reallocate()` function is the primary way a curated vault can remove liquidity from an underlying pool, so being unable to fully remove the liquidity is problematic. However, due to a logic issue in the implementation, any attempt to `reallocate()` liquidity in a pool to zero will revert. 

### Root Cause

The function `CuratedVault::reallocate()` is callable by an allocator and reallocates funds between underlying pools. As shown below, `toWithdraw` is the difference between `supplyAssets` (total assets in the underlying pool controlled by the curated vault) and the `allocation.assets` which is the target allocation for this pool. Therefore, if `toWithdraw` is greater than zero, a withdrawal is required from that pool:

```javascript
  function reallocate(MarketAllocation[] calldata allocations) external onlyAllocator {
    uint256 totalSupplied;
    uint256 totalWithdrawn;

    for (uint256 i; i < allocations.length; ++i) {
      MarketAllocation memory allocation = allocations[i];
      IPool pool = allocation.market;

      (uint256 supplyAssets, uint256 supplyShares) = _accruedSupplyBalance(pool);
@>    uint256 toWithdraw = supplyAssets.zeroFloorSub(allocation.assets);

      if (toWithdraw > 0) {
        if (!config[pool].enabled) revert CuratedErrorsLib.MarketNotEnabled(pool);

        // Guarantees that unknown frontrunning donations can be withdrawn, in order to disable a market.
        uint256 shares;
        if (allocation.assets == 0) {
          shares = supplyShares;
@>        toWithdraw = 0;
        }

@>      DataTypes.SharesType memory burnt = pool.withdrawSimple(asset(), address(this), toWithdraw, 0);
        emit CuratedEventsLib.ReallocateWithdraw(_msgSender(), pool, burnt.assets, burnt.shares);
        totalWithdrawn += burnt.assets;
      } else {
        
        ... SKIP!...
      }
    }
```

The issue arrises when for any given pool, `allocation.assets` is set to 0, meaning the allocator wishes to empty that pool and allocate the liquidity to another pool. Under this scenario, `toWithdraw` is set to 0, and passed into `Pool::withdrawSimple()`. This is a logic mistake to attempt to withdraw 0 instead of the `supplyAssets` when attempting to withdraw all liquidity to a pool. The call will revert due to a [check](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ValidationLogic.sol#L97) in the pool's withdraw flow that ensures the amount being withdrawn is greater than 0.

It seems this bug is a result of forking the Metamorpho codebase which [implements the same logic](https://github.com/morpho-org/metamorpho/blob/cc6b01610c9b0000d965faef705e1670859d9c0f/src/MetaMorpho.sol#L382-L389). However, Metamorpho's underlying withdraw() function [can take either an asset amount or share amount](https://github.com/morpho-org/morpho-blue/blob/0448402af51b8293ed36653de43cbee8d4d2bfda/src/Morpho.sol#L216-L217), but ZeroLend's `Pool::withdrawSimple()` only [accepts an amount of assets](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/Pool.sol#L94). 

### Internal pre-conditions

1. A curated vault having more than 1 underlying pool in the `supplyQueue`

### External pre-conditions

_No response_

### Attack Path

1. A curated vault is setup with more than 1 underlying pool in the `supplyQueue`
2. An allocator wishes to reallocate all the supplied liquidity from one pool to another
3. The allocator calls constructs the array of `MarketAllocation` withdrawing all liquidity from the first pool and depositing the same amount to the second pool
4. The action fails due to the described bug

### Impact

- Allocators cannot remove all liquidity in a pool throuhg the `reallocate()` function. The natspec comments indicate that emptying a pool to zero through the `reallocate()` function is the [first step of the intended way to remove a pool](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/interfaces/vaults/ICuratedVaultBase.sol#L141-L142).  

### PoC

Paste and run the below test in /test/forge/core/vaults/. It shows a simple 2-pool vault with a single depositors. An allocator attempts to reallocate the liquidity in the first pool to the second pool, but reverts due to the described issue.

```javascript
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity ^0.8.0;

import {Math} from '@openzeppelin/contracts/utils/math/Math.sol';
import './helpers/IntegrationVaultTest.sol';
import {console2} from 'forge-std/src/Test.sol';
import {MarketAllocation} from '../../../../contracts/interfaces/vaults/ICuratedVaultBase.sol';

contract POC_AllocatorCannotReallocateToZero is IntegrationVaultTest {
  function setUp() public {
    _setUpVault();
    _setCap(allMarkets[0], CAP);
    _sortSupplyQueueIdleLast();

    oracleB.updateRoundTimestamp();
    oracle.updateRoundTimestamp();
  }

  function test_POC_AllocatorCannotReallocateToZero() public {

    // 1. supplier supplies to underlying pool through curated vault
    uint256 assets = 10e18;
    loanToken.mint(supplier, assets);

    vm.startPrank(supplier);
    uint256 shares = vault.deposit(assets, onBehalf);
    vm.stopPrank();

    // 2. Allocator attempts to reallocate all of these funds to another pool
    MarketAllocation[] memory allocations = new MarketAllocation[](2);
    
    allocations[0] = MarketAllocation({
        market: vault.supplyQueue(0),
        assets: 0
    });

    // Fill in the second MarketAllocation struct
    allocations[1] = MarketAllocation({
        market: vault.supplyQueue(1),
        assets: assets
    });

    vm.startPrank(allocator);
    vm.expectRevert("NOT_ENOUGH_AVAILABLE_USER_BALANCE");
    vault.reallocate(allocations); // Reverts due to attempting a withdrawal amount of 0
  }
}
```

### Mitigation

The comment "Guarantees that unknown frontrunning donations can be withdrawn, in order to disable a market" does not seem to apply to Zerolend pools as a donation to the pool would not disable the market, nor would it affect the amount of assets the curated vault can withdraw from the underlying pool. With this in mind, Metamorpho's safety check can be removed:

```diff
  function reallocate(MarketAllocation[] calldata allocations) external onlyAllocator {
    uint256 totalSupplied;
    uint256 totalWithdrawn;


    for (uint256 i; i < allocations.length; ++i) {
      MarketAllocation memory allocation = allocations[i];
      IPool pool = allocation.market;

      (uint256 supplyAssets, uint256 supplyShares) = _accruedSupplyBalance(pool);
      uint256 toWithdraw = supplyAssets.zeroFloorSub(allocation.assets);

      if (toWithdraw > 0) {
        if (!config[pool].enabled) revert CuratedErrorsLib.MarketNotEnabled(pool);


-       // Guarantees that unknown frontrunning donations can be withdrawn, in order to disable a market.
-       uint256 shares;
-       if (allocation.assets == 0) {
-         shares = supplyShares;
-         toWithdraw = 0;
-       }

        DataTypes.SharesType memory burnt = pool.withdrawSimple(asset(), address(this), toWithdraw, 0);
        emit CuratedEventsLib.ReallocateWithdraw(_msgSender(), pool, burnt.assets, burnt.shares);
        totalWithdrawn += burnt.assets;
      } else {
        uint256 suppliedAssets =
          allocation.assets == type(uint256).max ? totalWithdrawn.zeroFloorSub(totalSupplied) : allocation.assets.zeroFloorSub(supplyAssets);

        if (suppliedAssets == 0) continue;

        uint256 supplyCap = config[pool].cap;
        if (supplyCap == 0) revert CuratedErrorsLib.UnauthorizedMarket(pool);

        if (supplyAssets + suppliedAssets > supplyCap) revert CuratedErrorsLib.SupplyCapExceeded(pool);

        // The market's loan asset is guaranteed to be the vault's asset because it has a non-zero supply cap.
        IERC20(asset()).forceApprove(address(pool), type(uint256).max);
        DataTypes.SharesType memory minted = pool.supplySimple(asset(), address(this), suppliedAssets, 0);
        emit CuratedEventsLib.ReallocateSupply(_msgSender(), pool, minted.assets, minted.shares);
        totalSupplied += suppliedAssets;
      }
    }

    if (totalWithdrawn != totalSupplied) revert CuratedErrorsLib.InconsistentReallocation();
  }
```