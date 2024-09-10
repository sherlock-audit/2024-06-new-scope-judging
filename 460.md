Scruffy Dijon Chinchilla

High

# `BNB` token can't be used if ZEROLEND deploys on `Linea` because chainlink oracle price feed doesn't support it

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