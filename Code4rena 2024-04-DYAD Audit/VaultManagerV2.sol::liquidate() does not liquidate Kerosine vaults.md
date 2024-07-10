## Confirmed: High

## Impact

Kerosine vaults are not counted for in `VaultManagerV2.sol::liquidate()` function which will lead to the function sending incorrect amount to liquidators causing a loss of funds for the liquidators. This leads will lead to liquidators avoiding calling `liquidate()` for users that keep part of their collateral in kerosine vaults, causing the protocol to accumulate bad debt and de-pegging the DYAD token.

## Proof of Concept

```solidity
    function liquidate(
        uint id,
        uint to
    ) external isValidDNft(id) isValidDNft(to) {
        uint cr = collatRatio(id);
        if (cr >= MIN_COLLATERIZATION_RATIO) revert CrTooHigh();
        dyad.burn(id, msg.sender, dyad.mintedDyad(address(this), id));

        uint cappedCr = cr < 1e18 ? 1e18 : cr;
        uint liquidationEquityShare = (cappedCr - 1e18).mulWadDown(
            LIQUIDATION_REWARD
        );
        uint liquidationAssetShare = (liquidationEquityShare + 1e18).divWadDown(
            cappedCr
        );

        uint numberOfVaults = vaults[id].length();
        for (uint i = 0; i < numberOfVaults; i++) {
            Vault vault = Vault(vaults[id].at(i));
            uint collateral = vault.id2asset(id).mulWadUp(
                liquidationAssetShare
            );
            vault.move(id, to, collateral);
        }
        emit Liquidate(id, msg.sender, to);
```

As we can see in the `liquidate()` function above, the function will loop([Lines 221-226](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L221-L226)) through the vaults to send liquidated users collateral to the liquidator. However we can see in the contracts mappings that vaults and kerosineVaults are stored separately ([Lines 34-35](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L34-L35)).

```solidity
  mapping (uint => EnumerableSet.AddressSet) internal vaults; 
  mapping (uint => EnumerableSet.AddressSet) internal vaultsKerosene; 
```

Since the liquidate function only looks for and their assets to liquidate and send to the liquidator and does not loop through the kerosene vault this will break the functions logic resulting in `liquidate()` function sending the liquidator less than the collateral amount they should get. 
This vulnerability disincentivizes liquidators which in turn will lead to bad debt accumulating in the contract and de-pegging the DYAD token.

## Tools Used

Manual review, foundry

## Recommended Mitigation Steps

Account for the keroseneVaults in the `liquidate()` function and update it as shown below:

```solidity
    function liquidate(
        uint id,
        uint to
    ) external isValidDNft(id) isValidDNft(to) {
        uint cr = collatRatio(id);
        if (cr >= MIN_COLLATERIZATION_RATIO) revert CrTooHigh();
        dyad.burn(id, msg.sender, dyad.mintedDyad(address(this), id));

        uint cappedCr = cr < 1e18 ? 1e18 : cr;
        uint liquidationEquityShare = (cappedCr - 1e18).mulWadDown(
            LIQUIDATION_REWARD
        );
        uint liquidationAssetShare = (liquidationEquityShare + 1e18).divWadDown(
            cappedCr
        );

        uint numberOfVaults = vaults[id].length();
        uint numberOfKeroseneVaults = keroseneVaults[id].lenght();
        for (uint i = 0; i < numberOfVaults; i++) {
            Vault vault = Vault(vaults[id].at(i));
            uint collateral = vault.id2asset(id).mulWadUp(
                liquidationAssetShare
            );
            vault.move(id, to, collateral);
        }
        for (uint i = 0; i < numberOfKeroseneVaults; i++) {
            Vault keroseneVault = Vault(keroseneVaults[id].at(i));
            uint collateral = keroseneVault.id2asset(id).mulWadUp(
                liquidationAssetShare
            );
            keroseneVault.move(id, to, collateral);
        }
        emit Liquidate(id, msg.sender, to);
```

.
