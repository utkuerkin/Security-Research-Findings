## Impact

Introduction of bounded Kerosine Vaults will break liquidation logic, will cause the protocol to accumulate bad debt and go underwater, will de-peg the Dyad token and tank the value of Kerosene tokens.

## Proof of Concept

First of we need to understand the difference of bounded kerosine compared to unbounded kerosine. This is explained in the  `Vault.kerosine.bounded.sol::assetPrice()` 

```solidity
  function assetPrice() 
    public 
    view 
    override
    returns (uint) {
      return unboundedKerosineVault.assetPrice() * 2;
  }
```

As we can see bounded kerosines are twice as more valuable as unbounded kerosines however they cannot be withdrawn because of the following code in `Vault.kerosine.bounded.sol::withdraw()`

```solidity
  function withdraw(
    uint    id,
    address to,
    uint    amount
  ) 
    external 
    view
      onlyVaultManager
  {
    revert NotWithdrawable(id, to, amount);
  }
```

Which essentially means that only use of bounded kerosine tokens are improving ones `collatRatio()`

```solidity
    function collatRatio(uint id) public view returns (uint) {
        uint _dyad = dyad.mintedDyad(address(this), id);
        if (_dyad == 0) return type(uint).max;
        return getTotalUsdValue(id).divWadDown(_dyad);
    }
    function getTotalUsdValue(uint id) public view returns (uint) {
        return getNonKeroseneValue(id) + getKeroseneValue(id);
    }
```

This allows users to mint more dyad in `VaultManagerV2.sol::mintDyad()` function. For example a user can deposit $101 worth of wETH and $24.5 worth of bounded Kerosene tokens to mint 100 DYAD tokens while without the bounded Kerosene they would need to deposit $150 worth of wETH as we can see in the `mintDyad()` function.

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

Problem starts when we look into the monetary gain of a liquidator when they call `VaultManagerV2.sol::liquidate()` function. With the exogeneous collateral vaults and unbounded kerosine vaults, liquidator will successfully get the amount they minted back and a 20% bonus.
In an example scenario where UserA is the liquidator and UserB is the user with the bad debt.

1. UserB’s collateral value goes down to $149 worth of tokens while they have 100 DYAD tokens minted and becomes eligible to be liquidated.
2. UserA calls `liquidate()` with UserB as the target.
3. UserA burns 100 DYAD in the liquidate function.
4. UserA will get approximately 73% of UserB’s tokens which equates to approximately $110 worth of tokens.
5. UserA has made approximately $10 profit from calling the `liquidate()` function and paying the bad debt of UserB.

However in a situation where UserB has used bounded kerosine tokens this changes, since UserB can mint 100 DYAD with $101 worth of exogeneous collateral and $24.5 worth of bounded kerosine tokens. Let’s explore this situation.

1. UserB’s exogeneous collateral value goes down to $100 worth of tokens while they have 100 DYAD tokens minted and becomes eligible to be liquidated.
2. UserA calls `liquidate()` with UserB as the target.
3. UserA burns 100 DYAD in the liquidate function.
4. UserA will get approximately 73% of UserB’s tokens which equates to approximately $73 worth of exogeneous collateral tokens and approximately $17.8 worth of bounded kerosine.

5. UserA has burnt 100 DYAD tokens while getting $73 worth of useable collateral and is currently in a deficit of $27 since bounded kerosine can not be withdrawn. 
Only way for UserA to make use of these bounded kerosine tokens is depositing more exogeneous collateral into the protocol to mint DYAD. At the current situation after liquidating, UserA can only mint 73 DYAD. In order to make use of the bounded collateral and make back the 100 DYAD they have burnt, UserA need to deposit approximately /$41 worth of exogeneous collateral making their total cost in this liquidation /$141 just to get back the 100 DYAD they have burnt.

Problem created with bounded kerosine vaults is that any user that has these tokens will never be profitable for a liquidator to liquidate. This would cause bad debt in the protocol to accumulate, therefor DYAD’s overcollateralization will keep decreasing which in turn tanks the Kerosine token value since it is tied to the overcollateralization of DYAD.

## Tools Used

Manual review, Foundry.

## Recommended Mitigation Steps

Do not use `Vault.kerosine.bounded.sol`
If bounded vault will be used, implement new ways to reward Liquidators that are getting bounded Kerosine as collateral to create incentive for liquidation.
