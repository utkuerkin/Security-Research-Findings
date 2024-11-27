## Confirmed: High

## Impact

Licensing of kerosene vaults are not handled correctly in the deployment script, this will lead to breaking the math of collaterals in the protocol leading to many critical issues.

## Proof of Concept

```solidity
    vaultLicenser.add(address(ethVault));
    vaultLicenser.add(address(wstEth));
    vaultLicenser.add(address(unboundedKerosineVault));
    // vaultLicenser.add(address(boundedKerosineVault));
```

[Lines 93-96](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/script/deploy/Deploy.V2.s.sol#L93-L96) in `Deploy.V2.s.sol` handles the licensing of the vaults which is an integral part of the protocol. However we can see that kerosine vaults are licensed with the `vaultLicenser` here aswell.

Vulnerability caused by this problem is that anyone can call the `VaultManagerV2.sol::add()` function for kerosine vaults aswell when that function should only work for exogeneous collateral vaults such as wETH. This leads to kerosine vaults being counted in `getNonKeroseneValue()` function shown below:

```solidity
    function getNonKeroseneValue(uint id) public view returns (uint) {
        uint totalUsdValue;
        uint numberOfVaults = vaults[id].length();
        for (uint i = 0; i < numberOfVaults; i++) {
            Vault vault = Vault(vaults[id].at(i));
            uint usdValue;
            if (vaultLicenser.isLicensed(address(vault))) {
                usdValue = vault.getUsdValue(id);
            }
            totalUsdValue += usdValue;
        }
        return totalUsdValue;
    }
```

effectively leading users kerosines to be counted as exogeneous collateral effectively doubling the value of kerosines counted as collateral and kerosines being counted in functions where they should not be counted such as `mintDyad()` function as shown below:

```solidity
  function mintDyad(
        uint id,
        uint amount,
        address to
    ) external isDNftOwner(id) {
        uint newDyadMinted = dyad.mintedDyad(address(this), id) + amount;
        if (getNonKeroseneValue(id) < newDyadMinted)
            revert NotEnoughExoCollat();
        dyad.mint(id, to, amount);
        if (collatRatio(id) < MIN_COLLATERIZATION_RATIO) revert CrTooLow();
        emit MintDyad(id, amount, to);
    }
```

In an example scenario where a user has 100$ worth of wETH as collateral and 100$ worth of unbounded kerosines they should be able to only mint a maximum of 100e18 DYAD in the intended use. With this vulnerability attacker will put the amount in `mintDyad()` as 200e18, `getNonKeroseneValue(id)` will return 200$ worth of tokens and will return false in the following if check `if (getNonKeroseneValue(id) < newDyadMinted)`. 200e18 DYAD will be minted to the attacker and the check for the next line: `if (collatRatio(id) < MIN_COLLATERIZATION_RATIO) revert CrTooLow();`. Here if we look at the `collatRatio()` function: 

```solidity
    function collatRatio(uint id) public view returns (uint) {
        uint _dyad = dyad.mintedDyad(address(this), id);
        if (_dyad == 0) return type(uint).max;
        return getTotalUsdValue(id).divWadDown(_dyad);
    }
```

We can see that it returns the `getTotalUsdValue(id).divWadDown(_dyad)` value. If we look into the `getTotalUsdValue()` function:

```solidity
function getTotalUsdValue(uint id) public view returns (uint) {
        return getNonKeroseneValue(id) + getKeroseneValue(id);
    }
```

We see that it is supposed to return values of collaterals in both vaults. However since kerosene vaults are also counted in `getNonKeroseneValue()` this function will return 200$ + 100$ worth of tokens and allowing the attacker to pass the `collatRatio()` check in `mintDyad()` resulting in the attacker being able to mint double the tokens that they should have minted.

## Tools Used

Manual review, foundry.

## Recommended Mitigation Steps

Instead of using `Licenser.sol` for kerosene vaults, create a new contract that is `KeroseneLicenser.sol` which will only handle licensing of kerosene vaults. Then change the deploy script as shown below.

```solidity
    vaultLicenser.add(address(ethVault));
    vaultLicenser.add(address(wstEth));
    keroseneVaultLicenser.add(address(unboundedKerosineVault));
    // keroseneVaultLicenser.add(address(boundedKerosineVault));
```

.
