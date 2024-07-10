## Confirmed: Medium

## Impact

`burnDyad()` function lacking access control leads to DoS.
Attacker will frontrun any `redeemDyad()` function calls with a `burnDyad()` call to DoS the `redeemDyad()` call.

## Proof of Concept

```solidity
    function burnDyad(uint id, uint amount) external isValidDNft(id) {
        dyad.burn(id, msg.sender, amount);
        emit BurnDyad(id, amount, msg.sender);
    }
```

`VaultManagerV2.sol::burnDyad()` function lacks access control and an attacker will be able to call this function.
An example attack scenario is shown below:

1. A normal user calls `redeemDyad()` function with `amount` = all their balance(Lets say 100e18 for simplicity).
2. Attacker seeing this call will then call `burnDyad()` function for 1 DYAD (DYAD has 1e18 decimals so 1 DYAD does not have any value) for the normal users vault.
3. `burnDyad()` function will call `dyad.burn()` with paremeters being `id` = normal users dNFT id, `msg.sender` = the vault, `amount` = 1 

```solidity
  function burn(
      uint    id, 
      address from,
      uint    amount
  ) external 
      licensedVaultManager 
    {
      _burn(from, amount);
      mintedDyad[msg.sender][id] -= amount;
  }
```

1. Here `Dyad.sol::burn()` function will reduce `mintedDyad[msg.sender][id]` by 1.
2. Attackers frontrun call finished and normal users `redeemDyad()` function reaches the first line where it will call `burn()` with `amount` = 100e18, however since the attacker has reduced `mintedDyad[msg.sender][id]` by 1, normal users call with revert due to an underflow error (9.999…e18 - 100e18).

Attacker will permanently DoS any users `redeemDyad()` call with barely any cost as long as the user tries to redeem all their balances.

## Tools Used

Manual review.

## Recommended Mitigation Steps

Implement access control to `burnDyad()` function as shown below:

```solidity
function burnDyad(uint id, uint amount) external isValidDNft(id) isDNftOwner(id) {
        dyad.burn(id, msg.sender, amount);
        emit BurnDyad(id, amount, msg.sender);
    }
```

.
