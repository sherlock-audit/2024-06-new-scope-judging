Joyous Cedar Tortoise

High

# When bad debt is accumulated, the loss is not shared amongst all suppliers, instead the last to withdraw will experience a huge loss

### Summary

When bad debt is accumulated, it should be socialised amongst all suppliers.

The issue is that the protocol does not do this, instead only the last users to withdraw funds will feel the effects of the bad debt.

If a pool experiences bad debt, the first users to withdraw will experience 0 loss, while the last to withdraw will experience a severe loss.

### Root Cause
The [withdrawn](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L106) assets is calculated as `shares * liquidityIndex` which does not take into account bad debt

This means that even if bad debt accrues, the first users to withdraw will be able to withdraw their shares to assets at a good rate, leaving the last users with all the loss. 

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

**This attack path is shown in the PoC:**

1. User A supplies 50 ETH to the pool
2. User B supplies 50 ETH to the pool
3. A bad debt liquidation occurs, so the liquidator only repaid 64 ETH out of the 100 ETH
4. User A is still able to withdraw their 50 ETH
5. User B is left with less than 14 ETH able to be withdrawn

### Impact

Huge fund loss for the suppliers last to withdraw

Early withdrawers effectively steal from the late withdrawers

### PoC

Add the following test to `PoolLiquidationTests.t.sol`

```solidity
function test__BadDebtLiquidationIsNotSocialised() external {
    // Setup users
    address supplierOfTokenB = address(124343434);
    address liquidator = address(8888);
    _mintAndApprove(alice, tokenA, 1000 ether, address(pool)); 
    _mintAndApprove(bob, tokenB, 50 ether, address(pool)); 
    _mintAndApprove(supplierOfTokenB, tokenB, 50 ether, address(pool)); 
    _mintAndApprove(liquidator, tokenB, 100 ether, address(pool)); 
    console.log("bob balance before: %e", tokenB.balanceOf(bob));
    // alice supplies 134 ether of tokenA
    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 268 ether, 0); 
    vm.stopPrank();

    // bob supplies 50 ether of tokenB
    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, 50 ether, 0); 
    vm.stopPrank();

    // supplierOfTokenB supplies 50 ether of tokenB
    vm.startPrank(supplierOfTokenB);
    pool.supplySimple(address(tokenB), supplierOfTokenB, 50 ether, 0); 
    vm.stopPrank();

    // alice borrows 100 ether of tokenB
    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, 100 ether, 0); 
    vm.stopPrank();

    // Drops the collateral price to make the position liquidatable
    // the drop is large enough to make the position in bad debt
    oracleA.updateAnswer(5e7); 

    // liquidator liqudiates the position
    // note that he takes all of alice's collateral
    vm.startPrank(liquidator);
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, 64 ether);

    // bob sees that the position is in bad debt and quickly withdraws all that he supplied
    // bob does not experience any loss from the bad debt
    vm.startPrank(bob);
    pool.withdrawSimple(address(tokenB), bob, 50 ether, 0);

    // supplierOfTokenB tries to withdraw his funds but it reverts, since there is none left
    vm.startPrank(supplierOfTokenB);
    vm.expectRevert();
    pool.withdrawSimple(address(tokenB), supplierOfTokenB, 50 ether, 0);

    // the maximum amount supplierOfTokenB can withdraw is 13 ether of tokenB
    // since that is all that is left in the pool
    pool.withdrawSimple(address(tokenB), supplierOfTokenB, 13 ether, 0);

    // log the final state
    console.log("The following is the final state");

    // show that there is no more tokenB left in the pool after bob withdrew everything
    uint256 PoolBalanceOfB = tokenB.balanceOf(address(pool));
    console.log("Remaining balance of tokenB in the pool = %e", PoolBalanceOfB);

    // show that bob got back the 50 ether he deposited
    uint256 BobBalanceOfB = tokenB.balanceOf(bob);
    console.log("bob's balance of tokenB = %e", BobBalanceOfB);

    // show that supplierOfTokenB only got back 13 ether of tokenB
    uint256 SupplierBalanceOfB = tokenB.balanceOf(supplierOfTokenB);
    console.log("SupplierBalanceOfB balance of tokenB = %e", SupplierBalanceOfB);

    uint256 aliceCollateral = pool.getBalance(address(tokenA), alice, 0);
    console.log("aliceCollateral =%e ", aliceCollateral);
  }
```
**Console output:**

```bash
Ran 1 test for test/forge/core/pool/PoolLiquidationTests.t.sol:PoolLiquidationTest
[PASS] test__BadDebtLiquidationIsNotSocialised() (gas: 1089597)
Logs:
  The following is the final state
  Remaining balance of tokenB in the pool = 8.09523809523809524e17
  bob's balance of tokenB = 5e19
  SupplierBalanceOfB balance of tokenB = 1.3e19
  aliceCollateral =0e0 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.34ms (1.66ms CPU time)

Ran 1 test suite in 11.14ms (4.34ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Mitigation

Socialise the loss among all depositors