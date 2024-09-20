Acrobatic Rainbow Grasshopper

Medium

# Min/max price is not checked

## Summary
Min/max price is not checked
## Vulnerability Detail
Upon getting a price, we do not validate the minimum and max prices from Chainlink:
```solidity
function getAssetPrice(address reserve) public view override returns (uint256) {
    (, int256 price,, uint256 updatedAt,) = IAggregatorV3Interface(_reserves[reserve].oracle).latestRoundData();
    require(price > 0, 'Invalid price');
    require(block.timestamp <= updatedAt + 1800, 'Stale Price');
    return uint256(price);
  }
```
While Chainlink has removed the minimum and max prices for a lot of feeds, they are still available for quite a few. Note that the rules consider this a valid issue:
>Issues related to minAnswer and maxAnswer checks on Chainlink's Price Feeds are considered medium only if the Watson explicitly mentions the price feeds (e.g. USDC/ETH) that require this check.

Here are some feeds with min and max prices on Arbitrum which is a chain that the protocol will be deployed on based on the README (as it's fully EVM-compatible):
ETH / BTC - https://arbiscan.io/address/0x3c8F2d5af2e0F5Ef7C23A08DF6Ad168ece071D4b#readContract
ETH / USD - https://arbiscan.io/address/0x3607e46698d218B3a5Cae44bF381475C0a5e2ca7#readContract
SOL / USD - https://arbiscan.io/address/0x8C4308F7cbD7fB829645853cD188500D7dA8610a#readContract
## Impact
Min/max price is not checked
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/pool/PoolGetters.sol#L158-L163
## Tool used

Manual Review

## Recommendation
Implement min and max price checks