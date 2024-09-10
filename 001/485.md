Massive Glass Crane

Medium

# CuratedVault::Reserves shares not included in totalShares while calculating supplyAssets of the Vaults Position.

### Summary
In the `_supplyPool()` , inorder to calculate the `supplyAssets` of the Vault's position in a pool we are calling `pool.marketBalances(asset())` which in turn returns `totalSupplyAssets` and `totalSupplyShares`. 

But the function `totalSupplyShares` includes only the `c8300e73f4d751796daad3dadbae4d11072b3d79` part of the `shares` and the `Treasury`'s `shares` is not included but the `totalSupplyAssets` contains the assets allocated for both the parties.

This will result in the wrong calculation of `supplyAssets`.
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L128

### Root Cause

`pool.marketBalances(asset())` returns the `totalSupplyShares` which is the total shares owned by the `depositors` in the reserve without including  the `accruedToTreasuryShares` which is allocated for the `treasury`.
[code](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/PoolGetters.sol#L176-L185)
```solidity
  function marketBalances(address asset) public view returns (uint256, uint256, uint256, uint256) {
    DataTypes.ReserveSupplies storage supplies = _totalSupplies[asset];

    return (
      supplies.getSupplyBalance(_reserves[asset].liquidityIndex), //@audit includes the assets reserved for reserves also
      supplies.supplyShares, //@audit returns only the supplyShares but not the treasury shares `accruedToTreasuryShares`
      supplies.getDebtBalance(_reserves[asset].borrowIndex),
      supplies.debtShares
    );
  }
```

For allocating assets to the pools in the `supplyQueue` , `_supplyPool()` is called.
There , inorder to calculate the `supplyAssets` of the vaults position , this incorrect `Supplyshares` is used  and hence resulting in the incorrect amount of `supplyAssets`. 
[CuratedVaultSetters.sol#L127-L128](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/vaults/CuratedVaultSetters.sol#L127-L128)
```solidity
supplyShares.toAssetsUp(totalSupplyAssets, totalSupplyShares)
```
So the `actual supplyAssets` is less than the `calculated supplyAssets` ,  which will also affect the `toSupply` variable which in turn affects the assets `amount` to deposit in that pool.

(Note that `Reserveshares` are minted to reserves while updating the state of the pool using updateState() .But the assets are transferred and  `Reserveshares`are minused from totalSupplyShare only when someone calls withdraw().[code](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/PoolSetters.sol#L84)  )

### Internal pre-conditions
`Lenders` supply assets using the `curatedVaults`.

`SupplyPool` in the `supplyQueue` has shares allocated to the `Reserve`.


### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact
Users cant deposit using curatedVaults even though the assets held by the position hasnt reached the supplyCap of the respective allocated pools in the supplyQueue.

supplyPool will start reverting even though the actual assets held by the Vault has not reached the supplyCap of that pool.

### PoC
_No response_


### Mitigation

pool.marketBalances(asset()); should return the totalShares which includes the reserve.accruedToTreasuryShares