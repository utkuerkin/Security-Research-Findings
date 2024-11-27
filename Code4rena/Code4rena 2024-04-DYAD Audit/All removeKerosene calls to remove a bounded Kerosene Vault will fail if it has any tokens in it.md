## Confirmed: High

## Impact

When a user has received any bounded Kerosene tokens and has the bounded Kerosene vault added to their Note, It will be impossible for the user to remove the vault from their Note.

## Proof of Concept

```solidity
    function removeKerosene(uint id, address vault) external isDNftOwner(id) {
        if (Vault(vault).id2asset(id) > 0) revert VaultHasAssets();
        if (!vaultsKerosene[id].remove(vault)) revert VaultNotAdded();
        emit Removed(id, vault);
    }
```

`VaultManagerV2.sol::removeKerosene()` has the [following check](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L113): `if (Vault(vault).id2asset(id) > 0) revert VaultHasAssets();`. This check is put so users can not accidentally remove any vaults they have assets in.
However bounded Kerosene Vault has no ways of moving these tokens. Withdraw is not allowed because of the nature of the vault and `Vault.kerosine.sol::move()` has access control where only vault manager can access it: 

```solidity
  function move(
    uint from,
    uint to,
    uint amount
  )
    external
      onlyVaultManager
  {
    id2asset[from] -= amount;
    id2asset[to]   += amount;
    emit Move(from, to, amount);
  }
```

Meaning once a user has added the vault into their Note they won’t have any ways of calling `removeKerosene()` on it since it will always have assets and function will always revert at: `if (Vault(vault).id2asset(id) > 0) revert VaultHasAssets();`.
Considering `VaultManagerV2.sol`  has the following limitation on Kerosene vaults that can be added: 

```solidity
uint public constant MAX_VAULTS_KEROSENE = 5;
```

This poses a risk where a user might want to stop using bounded vaults and add new vaults into their Note. Only way user can remove their bounded kerosene from their account and be able to remove the bounded Kerosene vault is if they get fully liquidated.

## Tools Used

Manual review, Foundry.

## Recommended Mitigation Steps

Implement a new `move()` function for `Vault.kerosine.bounded.sol` or remove the access control in `Vault.kerosine.sol::move()`.
