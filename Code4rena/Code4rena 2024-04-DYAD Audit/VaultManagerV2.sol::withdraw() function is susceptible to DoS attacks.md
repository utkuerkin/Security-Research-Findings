## Confirmed: High

## Impact

`deposit()` function lacking access control leads to DoS. 
Attacker will frontrun any `withdraw()` call with a `deposit()` call to DoS the `withdraw()` call, causing it to revert.

## Proof of Concept

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

Shown above is the `VaultManagerV2.sol::withdraw()` function which has a flash loan protection in the code snippet shown below:

```solidity
 if (idToBlockOfLastDeposit[id] == block.number)
            revert DepositedInSameBlock();
```

However if we look at where `idToBlockOfLastDeposit[id]` is updated, `VaultManagerV2.sol::deposit()` function is the one that updates `idToBlockOfLastDeposit[id]` 

```solidity
function deposit(
        uint id,
        address vault,
        uint amount
    ) external isValidDNft(id) {
        idToBlockOfLastDeposit[id] = block.number;
        Vault _vault = Vault(vault);
        _vault.asset().safeTransferFrom(msg.sender, address(vault), amount);
        _vault.deposit(id, amount);
    }
```

Problem here is that `deposit()` function lacks the `isDNftOwner(id)` modifier, meaning anyone can call this function with 1 token to update that dNFT’s `idToBlockOfLastDeposit[id]`.

Example attack happens like this:
1. A normal user calls `withdraw()` to withdraw their collateral from their vault.
2. Attacker will see this call and frontrun it with their own `deposit()` call sending it in the same block, depositing 1 token into the normal users vault. (wETH and wstETH are 18 decimal tokens so 1 token is not worth anything.)
3. `deposit()` call will update the `idToBlockOfLastDeposit[id]` with the current `block.number`.

1. Normal users `withdraw()` call will revert when it reaches the `if (idToBlockOfLastDeposit[id] == block.number)` check because block.number will be the same.

Since this attack costs almost nothing to the attacker, attacker will keep frontrunning to permanently DoS the user.

## Tools Used

Manual review.

## Recommended Mitigation Steps

Add the `isDNftOwner(id)` modifier in the `deposit()` function as shown below:

```solidity
function deposit(
        uint id,
        address vault,
        uint amount
    ) external isValidDNft(id) isDNftOwner(id) {
        idToBlockOfLastDeposit[id] = block.number;
        Vault _vault = Vault(vault);
        _vault.asset().safeTransferFrom(msg.sender, address(vault), amount);
        _vault.deposit(id, amount);
    }
```

.
