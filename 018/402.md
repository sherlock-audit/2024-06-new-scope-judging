Clever Ebony Halibut

Medium

# Inconsistent Application of Reserve Factor Changes Leads to Protocol Insolvency Risk

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