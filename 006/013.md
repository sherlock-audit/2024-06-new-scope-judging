Sleepy Rose Albatross

Medium

# Usage of transfer in `sweep` can result in revert

## Summary
In `NFTPositionManager::sweep` , transfer is being used in case of eth, which will result revert in case Admin is multisig or smart wallet.

## Vulnerability Detail

Relevant link  - https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManager.sol#L131-L139
When using transfer function for sending eth to an address it uses a fixed gas of 2300. It works fine for normal E0A. 
But it fails, when receiver is multisig wallet (aka safe wallet) or a smart wallet that mostly use more than 2300 gas. 
To be precise, it's gonna revert when 
-  receiver smart contract (smart wallet) implements a payable fallback, which needs more than 2300 gas.
-  receiver smart contract (aka multisig, safe) that  implements the payable fallback function which needs less than 2300 gas units but is called through a proxy that raises the call’s gas usage above 2300.
- receiver don't have payable fallback implemented.

## Impact

transfer() uses a fixed amount of gas, which can result in revert. https://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/

## Code Snippet

```solidity
function sweep(address token) external onlyRole(DEFAULT_ADMIN_ROLE) {
    if (token == address(0)) {
      uint256 bal = address(this).balance;
@>      payable(msg.sender).transfer(bal);
    } else {
      IERC20Upgradeable erc20 = IERC20Upgradeable(token);
      erc20.safeTransfer(msg.sender, erc20.balanceOf(address(this)));
    }
  }
```

## Tool used

Manual Review

## Recommendation

using call will be much better option.

```solidity
function sweep(address token) external onlyRole(DEFAULT_ADMIN_ROLE) {
    if (token == address(0)) {
      uint256 bal = address(this).balance;

    (bool sent,) = (msg.sender).call{value: bal}(""):
      if(!sent){revert EthTransferFailed()):
    } else {
      IERC20Upgradeable erc20 = IERC20Upgradeable(token);
      erc20.safeTransfer(msg.sender, erc20.balanceOf(address(this)));
    }
  }

