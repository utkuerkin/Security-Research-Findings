## Confirmed: High

## Impact

Unbounded Kerosine tokens are withdrawable in the intended use of the protocol. However a vulnerability in `VaultManagerV2.sol` causes them to be stuck in the contract. Difference of unbounded and bounded kerosine tokens are that bounded ones count as twice the value when `collatRatio()` of a user is being calculated with the downside of not being able to withdraw them, this vulnerability causes unbounded tokens to have the same downside while not having the upside of counting double which will lead to users tokens being forever stuck in the vault.
Same vulnerability causes `redeemDyad()` function to revert as well.

## Proof of Concept

If we look at the `Vault.kerosine.unbounded.sol::withdraw()`: 

```solidity
    function withdraw(uint id, address to, uint amount) external onlyVaultManager {
        id2asset[id] -= amount;
        asset.safeTransfer(to, amount);
        emit Withdraw(id, to, amount);
    }
```

we can see that it is only meant to be called by the `VaultManagerV2.sol` because of the `onlyVaultManager` modifier found in `Vault.kerosine.sol`. 

```solidity
  modifier onlyVaultManager() {
    if (msg.sender != address(vaultManager)) revert NotVaultManager();
    _;
  }
```

This means only way a user can withdraw their unbounded kerosine tokens is calling the `withdraw()` function in `VaultManagerV2.sol`. 

```solidity
function withdraw(
        uint id,
        address vault,
        uint amount,
        address to
    ) public isDNftOwner(id) {
        if (idToBlockOfLastDeposit[id] == block.number)
            revert DepositedInSameBlock();
        uint dyadMinted = dyad.mintedDyad(address(this), id);
        Vault _vault = Vault(vault);
        uint value = (amount * _vault.assetPrice() * 1e18) / 10 ** _vault.oracle().decimals() / 10 ** _vault.asset().decimals();
        if (getNonKeroseneValue(id) - value < dyadMinted)
            revert NotEnoughExoCollat();
        _vault.withdraw(id, to, amount);
        if (collatRatio(id) < MIN_COLLATERIZATION_RATIO) revert CrTooLow();
    }
```

However when a user tries to withdraw their kerosene token, entering their kerosene token vault as the address, following line in `withdraw()` will try to calculate the worth of the kerosene tokens in this line: `uint value = (amount * _vault.assetPrice() * 1e18) / 10 ** _vault.oracle().decimals() / 10 ** _vault.asset().decimals();`. Due to a lack of `oracle` in `Vault.kerosine.unbounded.sol` the transaction will fail when the function tries to calculate the value.
This vulnerability also causes `withdraw()` calls to `Vault.kerosine.bounded.sol` to revert with the incorrect error.

Same vulnerability will happen in `redeemDyad()` function as well: 

```solidity
    function redeemDyad(
        uint id,
        address vault,
        uint amount,
        address to
    ) external isDNftOwner(id) returns (uint) {
        dyad.burn(id, msg.sender, amount);
        Vault _vault = Vault(vault);
        uint asset = (amount * (10 ** (_vault.oracle().decimals() + _vault.asset().decimals()))) / _vault.assetPrice() / 1e18;
        withdraw(id, vault, asset, to);
        emit RedeemDyad(id, vault, amount, to);
        return asset;
    }
```

Kerosene vaults not having an oracle will cause this function to revert.

## Tools Used

Manual review, Foundry.

## Recommended Mitigation Steps

Implement a new `withdrawUnboundedKerosene()` function in `VaultManagerV2.sol` 

```solidity
function withdrawUnboundedKerosene(
        uint id,
        address vault,
        uint amount,
        address to
    ) public isDNftOwner(id) {
        if (idToBlockOfLastDeposit[id] == block.number)
            revert DepositedInSameBlock();
        uint dyadMinted = dyad.mintedDyad(address(this), id);
        Vault _vault = Vault(vault);
        uint value = (amount * _vault.assetPrice() / 10 ** _vault.asset().decimals();
        _vault.withdraw(id, to, amount);
        if (collatRatio(id) < MIN_COLLATERIZATION_RATIO) revert CrTooLow();
    }
```

Implement a new `redeemDyadWithKerosene()` function in `VaultManagerV2.sol` 

```solidity
    function redeemDyadWithKerosene(
        uint id,
        address vault,
        uint amount,
        address to
    ) external isDNftOwner(id) returns (uint) {
        dyad.burn(id, msg.sender, amount);
        Vault _vault = Vault(vault);
        uint value = (amount * _vault.assetPrice() / 10 ** _vault.asset().decimals();
        withdraw(id, vault, asset, to);
        emit RedeemDyad(id, vault, amount, to);
        return asset;
    }
```

.
