Dazzling Sand Griffin

Medium

# ownership is not deployed properly for CuratedVaultFactory.sol & poolFactory.sol

## Summary
ownable's constructor isn't called by CuratedVaultFactory.sol & poolFactory.sol's constructor.

## Vulnerability Detail
Both CuratedVaultFactory.sol & poolFactory.sol inherit OZ's Ownable2Step.sol BUT fails to properly deploy it.

OZ's Ownable2Step.sol relies on ownable's constructor to have an initialOwner because it inherits ownable.sol
https://github.com/OpenZeppelin/openzeppelin-contracts/blob/d8bbd346762c7bf9f594777dfab788d214423ced/contracts/access/Ownable2Step.sol#L6

https://github.com/OpenZeppelin/openzeppelin-contracts/blob/d8bbd346762c7bf9f594777dfab788d214423ced/contracts/access/Ownable.sol#L20

The issue here is that CuratedVaultFactory.sol & poolFactory.sol's constructors fail to call Ownable.sol's constructor 
```solidity
   /**
     * @dev Initializes the contract setting the deployer as the initial owner.
     */
    constructor() {
        _transferOwnership(_msgSender());
    }
```

Now since they don't call Ownable.sol's constructor, the contracts are without an owner
```solidity
 constructor(address _implementation) {//@audit ownership is not deployed properly for CVF & pool Factory {ownable() is not called here}
    implementation = _implementation;
  }

```

```solidity
 constructor(address _implementation) {
    implementation = _implementation;
    treasury = msg.sender;
  }
```
## Impact
CuratedVaultFactory.sol & poolFactory.sol will be without an owner and functions with `onlyOwner` modifier will be uncallable.
## Code Snippet
https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/PoolFactory.sol#L58

https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/CuratedVaultFactory.sol#L38
## Tool used

Manual Review

## Recommendation
do this(i.e call ownable's constructor in their constructors):
```solidity
constructor(address _implementation) Ownable() {
```
