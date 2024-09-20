Chilly Cherry Deer

Medium

# Incorrect cap on boosted balance in `boostedBalance` Function

### Summary

The incorrect cap logic in the `boostedBalance` function will cause an unintended limitation on reward boosts for users as the contract will cap the boosted balance to the original balance, preventing users from receiving the full potential boost of up to 5x.
- [NFTRewardsDistributor.sol#L122](https://github.com/sherlock-audit/2024-06-new-scope/blob/c8300e73f4d751796daad3dadbae4d11072b3d79/zerolend-one/contracts/core/positions/NFTRewardsDistributor.sol#L122)

```solidity
return _boostedBalance > balance ? balance : _boostedBalance;
```

##### Example
- User's Original Balance: 100 tokens
- User's Stake: Results in a calculated boost that should be 5x the original balance.
- Expected Boosted Balance: 500 tokens (5x the original balance)
- Current logic
```solidity
 // because of this we are able to max out the boost by 5x
    uint256 _boostedBalance = _boosted + _adjusted;
    return _boostedBalance > balance ? balance : _boostedBalance;
```
- The comment in the code suggests that the boost can be up to 5x, implying that the boosted balance should potentially be up to five times the original balance, depending on the user's stake.
- Capped Boosted Balance: The logic return `_boostedBalance > balance ? balance : _boostedBalance;`
 will cap the boosted balance to the original balance of 100 tokens.`

### Root Cause

In the `boostedBalance` function, the current logic to cap the boosted balance to the original balance is a mistake as it prevents users from receiving the intended 5x boost based on their stake, which undermines the incentive mechanism.


### Impact

The users cannot receive the full potential boost of up to 5x on their staked balance, limiting their rewards and undermining the intended incentive structure. 


### Mitigation

- remove cap
- or Implement logic to ensure the boosted balance does not exceed this maximum allowed boost