## Confirmed: High

## Impact

Kerosene vault licensing is handled incorrectly, vulnerability this causes is that exogenous collateral vaults such as wETH vault will also be licensed as kerosene vaults allowing an attacker to double their `collatRatio()`, minting more tokens than they should be allowed which will lead to de-pegging of DYAD token.

## Proof of Concept

If we look at `Deploy.V2.s.sol` following lines([64-65](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/script/deploy/Deploy.V2.s.sol#L64-L65)) add exogenous vaults into the `KerosineManager.sol` 

```solidity
kerosineManager.add(address(ethVault));
kerosineManager.add(address(wstEth));
```

This is done because the protocol wants to calculate the TVL in order to calculate the Kerosene asset price in the `Vault.kerosine.unbounded.sol::assetPrice()` function.

```solidity
    function assetPrice() public view override returns (uint) {
        uint tvl;
        address[] memory vaults = kerosineManager.getVaults();
        uint numberOfVaults = vaults.length;
        for (uint i = 0; i < numberOfVaults; i++) {
            Vault vault = Vault(vaults[i]);
            tvl += (vault.asset().balanceOf(address(vault)) * vault.assetPrice() * 1e18) / (10 ** vault.asset().decimals()) / (10 ** vault.oracle().decimals());
        }
        uint numerator = tvl - dyad.totalSupply();
        uint denominator = kerosineDenominator.denominator();
        return (numerator * 1e8) / denominator;
    }
```

This function will loop through the added exogenous vaults to get their values and calculate `tvl`.

If we look into how Kerosene vaults are added into `VaultManagerV2.sol` : 

```solidity
    function addKerosene(uint id, address vault) external isDNftOwner(id) {
        if (vaultsKerosene[id].length() >= MAX_VAULTS_KEROSENE)
            revert TooManyVaults();
        if (!keroseneManager.isLicensed(vault)) revert VaultNotLicensed();
        if (!vaultsKerosene[id].add(vault)) revert VaultAlreadyAdded();
        emit Added(id, vault);
    }
```

we see that function checks if the kerosene vaults are licensed in this line: `if (!keroseneManager.isLicensed(vault)) revert VaultNotLicensed();`. `KerosineManager.sol` checks for the license in the following function: 

```solidity
  function isLicensed(
    address vault
  ) 
    external 
    view 
    returns (bool) {
      return vaults.contains(vault);
  }
```

Since the deploy script added exogenous collateral vaults into the `KerosineManager.sol` contract this automatically means these vaults are also licensed by the KerosineManager.

An attack scenario happens like this:

1. Attacker will use this vulnerability to call `VaultManagerV2.sol::add()` and `VaultManagerV2.sol::addKerosene()` functions with wETH vault adding the same vault in seperate mappings. 

```solidity
mapping(uint => EnumerableSet.AddressSet) internal vaults;
mapping(uint => EnumerableSet.AddressSet) internal vaultsKerosene;
```

1. Attacker will deposit \$100 worth of wETH into the wETH vault.
2. Attacker will call the `mintDyad()` function with `amount` = 99e18. 

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

1. `if (getNonKeroseneValue(id) < newDyadMinted)` check will be false and not revert (100e18<99e18)
2. Function will call `dyad.mint()` and mint 99 DYAD tokens.
3. Function will check the following line: `if (collatRatio(id) < MIN_COLLATERIZATION_RATIO) revert CrTooLow();` 

```solidity
   function collatRatio(uint id) public view returns (uint) {
        uint _dyad = dyad.mintedDyad(address(this), id);
        if (_dyad == 0) return type(uint).max;
        return getTotalUsdValue(id).divWadDown(_dyad);
    }
    
    function getTotalUsdValue(uint id) public view returns (uint) {
        return getNonKeroseneValue(id) + getKeroseneValue(id);
    }
    
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

    function getKeroseneValue(uint id) public view returns (uint) {
        uint totalUsdValue;
        uint numberOfVaults = vaultsKerosene[id].length();
        for (uint i = 0; i < numberOfVaults; i++) {
            Vault vault = Vault(vaultsKerosene[id].at(i));
            uint usdValue;
            if (keroseneManager.isLicensed(address(vault))) {
                usdValue = vault.getUsdValue(id);
            }
            totalUsdValue += usdValue;
        }
        return totalUsdValue;
    }
```

1. Here we see that `collatRatio()` checks the `getTotalUsdValue()` function. Since the attacker has added exogenous collateral vaults as both normal vaults and kerosene vaults, both `getNonKeroseneValue()` and `getKeroseneValue()` functions will return 100e18, making total USD value 200e18.
2. `collatRatio()` will return 2e18 to the `mintDyad()` function, making the `if (collatRatio(id) < MIN_COLLATERIZATION_RATIO)` return false(2e18 < 1.5e18) therefor it will not revert and transaction will complete successfully.The attacker now has minted 99e18 DYAD with only \$100 worth of collateral when in reality they should need \$200 worth of collateral to mint.

## Tools Used

Manual review, Foundry

## Recommended Mitigation Steps

Change the licensing of kerosene vaults. An example is shown below:
1. Create a new contract called `KeroseneLicenser.sol` that handles licensing of Kerosene vaults.

1. Delete `KerosineManager.sol::isLicensed()` function.
2. Change all `keroseneManager.isLicensed(vault)` lines in `VaultManagerV2.sol` into `keroseneLicenser.isLicensed(vault)`.
