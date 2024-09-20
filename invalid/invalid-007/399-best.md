Clever Ebony Halibut

Medium

# Non-Compliant `maxWithdraw` and `maxRedeem` Functions in CuratedVault Implementation

## Summary

The `maxWithdraw` function in the `CuratedVault` contract does not comply with the ERC-4626 standard. It incorrectly calculates the available liquidity for withdrawals, potentially returning a value that exceeds the actual maximum withdrawable amount and causing a revert when users attempt to withdraw the suggested maximum.

## Vulnerability Detail

- sherlock rules :

> The protocol team can use the README (and only the README) to define language that indicates the codebase's restrictions and/or expected functionality. Issues that break these statements, irrespective of whether the impact is low/unknown, will be assigned Medium severity. High severity will be applied only if the issue falls into the High severity category in the judging guidelines.

- from the readme :

> The CuratorVault should follow the ERC4626 standard.

The contract's `maxWithdraw` function doesn't comply with `ERC-4626` which is a mentioned requirement.
According to the eip specification:

> MUST return the maximum amount of assets that could be transferred from owner through withdraw and not cause a revert, which MUST NOT be higher than the actual maximum that would be accepted (it should underestimate if necessary)

- however the `maxWithdraw` function in `CuratedVault` can return a value that exceeds the actual maximum and cause a revert as we see in the function :

```js
  function maxWithdraw(address owner) public view override returns (uint256 assets) {
    (assets,,) = _maxWithdraw(owner);
  }
    function _maxWithdraw(address owner) internal view returns (uint256 assets, uint256 newTotalSupply, uint256 newTotalAssets) {
    uint256 feeShares;
    (feeShares, newTotalAssets) = _accruedFeeShares();
    newTotalSupply = totalSupply() + feeShares;

    assets = _convertToAssetsWithTotals(balanceOf(owner), newTotalSupply, newTotalAssets, MathUpgradeable.Rounding.Down);
    assets -= _simulateWithdraw(assets);
  }
    function _simulateWithdraw(uint256 assets) internal view returns (uint256) {
    for (uint256 i; i < withdrawQueue.length; ++i) {
      IPool pool = withdrawQueue[i];
      DataTypes.PositionBalance memory pos = pool.getBalanceRawByPositionId(asset(), positionId);
      uint256 supplyShares = pos.supplyShares;
      (uint256 totalSupplyAssets, uint256 totalSupplyShares, uint256 totalBorrowAssets,) = pool.marketBalances(asset());

      // The vault withdrawing from ZeroLend cannot fail because:
      // 1. oracle.price() is never called (the vault doesn't borrow)
      // 2. the amount is capped to the liquidity available on ZeroLend
      // 3. virtually accruing interest didn't fail
  >>  assets = assets.zeroFloorSub(_withdrawable(pool, totalSupplyAssets, totalBorrowAssets, supplyShares.toAssetsDown(totalSupplyAssets, totalSupplyShares)));

      if (assets == 0) break;
    }

    return assets;
  }
```

- the function try to simulate the withdraw by calling the `_withdrawable` function which is defined as :

```js
    function _withdrawable(IPool pool, uint256 totalSupplyAssets, uint256 totalBorrowAssets, uint256 supplyAssets) internal view returns (uint256) {
    // Inside a flashloan callback, liquidity on the pool may be limited to the singleton's balance.
 >>   uint256 availableLiquidity = UtilsLib.min(totalSupplyAssets - totalBorrowAssets, IERC20(asset()).balanceOf(address(pool)));
    return UtilsLib.min(supplyAssets, availableLiquidity);
  }
```

- the issue here is that the `_withdrawable` function caps the amount of assets that can be withdrawn by the vault to the available liquidity on the pool using `balanceOf` . and assumes that the whole balance can be withdrawn by the vault (with respect to it's shares and user shares). however this is not the case , as the available liquidity should be enough for : **vault withdrawal + treasury withdrawal** as we see in the withdraw function :

```js

  function _withdraw(/*params*/ ) internal nonReentrant(RentrancyKind.LENDING) returns (DataTypes.SharesType memory res) {
    require(to != address(0), PoolErrorsLib.ZERO_ADDRESS_NOT_VALID);
    DataTypes.ExecuteWithdrawParams memory params =
      DataTypes.ExecuteWithdrawParams({reserveFactor: _factory.reserveFactor(), asset: asset, amount: amount, position: pos, destination: to, data: data, pool: address(this)});

    if (address(_hook) != address(0)) _hook.beforeWithdraw(params);

 >> res = SupplyLogic.executeWithdraw(_reserves, _reservesList, _usersConfig[pos], _balances, _totalSupplies[asset], params);
    // send amount accrue to treasury , and update the share supply and reset treasury accrue to 0
 >> PoolLogic.executeMintToTreasury(_totalSupplies[asset], _reserves, _factory.treasury(), asset);

    if (address(_hook) != address(0)) _hook.afterWithdraw(params);
  }
```

### example poc :

Let's assume the following scenario:

1. Total supply assets in the pool: 1,000,000 USDC
2. Total borrow assets in the pool: 800,000 USDC
3. Pool's USDC balance: 105,000 USDC
4. Vault's supply assets: 150,000 USDC
5. Treasury's accrued fees: 5,000 USDC

In this case:

- Available liquidity according to `_withdrawable()`:
  $min(1,000,000 - 800,000, 105,000) = 105,000 USDC$

- The vault assumes it can withdraw up to 105,000 USDC

However, the actual maximum withdrawal amount should be:
$105,000 - 5,000 = 100,000 USDC$

This is because 5,000 USDC needs to be reserved for the treasury's fee withdrawal.

- If the vault attempts to withdraw its full 105,000 USDC, the transaction will revert due to insufficient balance to send to treasury .

## Impact

Failure to comply with the ERC-4626 specification, as expected for the vault contract.

## Code Snippet

- https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultGetters.sol#L65-L67

## Tool used

Manual Review

### Recommendation

To address this issue, modify the `_withdrawable` function to account for the treasury's accrued fees: