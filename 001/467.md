Long Coffee Cat

High

# NFTPositionManager's `repay()` and `repayETH()` are unavailable unless preceded atomically by an accounting updating operation

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

