Great Jade Shetland

Medium

# Using the same heartbeat for multiple price feeds, causing DOS

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