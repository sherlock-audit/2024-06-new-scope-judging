Happy Midnight Griffin

High

# Liquidation can be DOSed due to lack of liquidity on collateral asset reserve

### Summary

Lack of liquidity on collateral asset reserves can cause disruption to liquidation. 

### Root Cause

The protocol don't have option to disable borrowing or withdrawing in a particular asset reserve for a certain extent to protect the collateral deposits. It can only [disable borrowing](https://github.com/sherlock-audit/2024-06-new-scope-bluenights004/blob/main/zerolend-one/contracts/core/pool/configuration/ReserveConfiguration.sol#L166-L167) or [freeze](https://github.com/sherlock-audit/2024-06-new-scope-bluenights004/blob/main/zerolend-one/contracts/core/pool/configuration/ReserveConfiguration.sol#L148-L149) the whole reserves but not for specific portion such as collateral deposits. This can be a big problem because someone can always deduct or empty the reserves either by withdrawing their lended assets or borrowing loan. And when the liquidation comes, the collateral can't be paid to liquidator because the asset reserve is already not enough or emptied.

The pool administrator might suggest designating whole asset reserve to be used only for collateral deposit purposes and not for lending and borrowing. However this can be circumvented by malicious users by transferring their collateral to other reserves that accepts borrowing and lending which can eventually led the collateral to be borrowed. This is the nature of multi-asset lending protocol, it allows multiple asset reserves for borrowing and lending as per protocol documentation.

There could be another suggestion to resolve this by only using one asset reserve per pool that offers lending and borrowing but this will already contradict on what the protocol intends to be which is to be a multi-asset lending pool, meaning there are multiple assets offering lending in single pool.

If the protocol intends to do proper multi-asset lending pool platform, it should protect the collateral assets regarding liquidity issues. 

### Internal pre-conditions

1. Pool creator should setup the pool with more than 2 asset reserves offering lending or borrowing and each of reserves accepts collateral deposits. It allows any of the asset reserves to conduct borrowing to any other asset reserves and vice versa. This is pretty much the purpose and design of the multi-asset lending protocol as per documentation.


### External pre-conditions

_No response_

### Attack Path

This can be considered as attack path or can happen also as normal scenario due to the nature or design of the multi-asset lending protocol. Take note the step 6 can be just a normal happening or deliberate attack to disrupt the liquidation.

<img width="579" alt="image" src="https://github.com/user-attachments/assets/3a33a2b1-1e9d-4cc5-8463-8b4a4fcc5f46">



### Impact
This should be high risk since in a typical normal scenario, this vulnerability can happen without so much effort.
The protocol also suffers from bad debt as the loan can't be liquidated.

### PoC

1. Modify this test file /zerolend-one/test/forge/core/pool/PoolLiquidationTests.t.sol
and insert the following:
a. in line 16, put address carl = address(3); // add carl as borrower
b.  modify this function _generateLiquidationCondition() internal {
    _mintAndApprove(alice, tokenA, mintAmountA, address(pool)); // alice 1000 tokenA
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool)); // bob 2000 tokenB
    _mintAndApprove(carl, tokenB, mintAmountB, address(pool)); // carl 2000 tokenB >>> add this line 


    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, supplyAmountA, 0); // 550 tokenA alice supply
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0); // 750 tokenB bob supply
    vm.stopPrank();

    vm.startPrank(carl);
    pool.supplySimple(address(tokenB), carl, supplyAmountB, 0); // 750 tokenB carl supply >>> add this portion
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, borrowAmountB, 0); // 100 tokenB alice borrow
    vm.stopPrank();

    assertEq(tokenB.balanceOf(alice), borrowAmountB);

    oracleA.updateAnswer(5e3);
  }
  c. Insert this test
  function testLiquidationSimple2() external {
    _generateLiquidationCondition();
    (, uint256 totalDebtBase,,,,) = pool.getUserAccountData(alice, 0);

    vm.startPrank(carl);
    pool.borrowSimple(address(tokenA), carl, borrowAmountB, 0); // 100 tokenA carl borrow to deduct the reserves in which the collateral is deposited
    vm.stopPrank();

    vm.startPrank(bob);
    vm.expectRevert();
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, 10 ether); // Bob tries to liquidate Alice but will revert

    vm.stopPrank();

    (, uint256 totalDebtBaseNew,,,,) = pool.getUserAccountData(alice, 0);

    // Ensure that no liquidation happened and Alice's debt remains the same
    assertEq(totalDebtBase, totalDebtBaseNew, "Debt should remain the same after failed liquidation");

  }
  2. Run the test forge test -vvvv --match-contract PoolLiquidationTest --match-test testLiquidationSimple2

### Mitigation

Each asset reserve should be modified to not allow borrowing or withdrawing for certain collateral deposits. For example, if a particular asset reserve has deposits for collateral, these deposits should not be allowed to be borrowed or withdrew. The rest of the balance of asset reserves will do the lending. At the current design, the pool admin can only make the whole reserve as not enabled for borrowing but not for specific account or amount.