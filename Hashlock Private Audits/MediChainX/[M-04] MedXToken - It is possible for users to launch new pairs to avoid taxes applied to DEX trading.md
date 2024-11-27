## Description
MedXToken has a 3% fee built in on DEX trades (i.e buying or selling on UniSwapV2
pairs.) The addresses that the tax will apply to are added manually, meaning that the
protocol might forget to add pairs to be taxed, users would prefer to trade on the pairs
that are not yet taxed or users would launch more and more pairs, turning it into a race.

## Vulnerability Details
The `constructor` in the contract creates two new pairs for the token and adds them to
the `applyTax` mapping.
```solidity
constructor(
    address _owner,
    address payable _feeReceiver,
    IUniswapV2Router02 _uniV2Router,
    address usdt
) ERC20("MedXT", "$MedXT") Ownable(_owner) {
    //rest of the constructor
    address wethUniV2Pair = factory.createPair(self, weth);
    address usdtUniV2Pair = factory.createPair(self, usdt);
    _updateTaxedList(wethUniV2Pair, true);
    _updateTaxedList(usdtUniV2Pair, true);
    _approve(self, address(_uniV2Router), type(uint256).max);
}
```
When the token is launched only these two pairs will have tax applied to them. Take a
look at the `_update` function below.
```solidity
function _update(address from, address to, uint256 value) internal override(ERC20) {
    if (blacklisted[from]) revert Blacklisted(from);
    if (blacklisted[to]) revert Blacklisted(to);
    if (
        !feeEnabled ||
        whitelisted[from] ||
        whitelisted[to] ||
        from == address(this) ||
        to == address(this)
    ) {
        return super._update(from, to, value);
    }
    if (applyTax[to]) _updateWithSellFee(from, to, value);
    else if (applyTax[from]) _updateWithBuyFee(from, to, value);
    else super._update(from, to, value);
}
```
It is observed that the tax is applied to only the addresses in the applyTax mapping.
There are no dynamic checks to gure out if the to or from addresses are DEX pairs
which will lead to users launching new pairs to avoid taxes. For example, the
MedX-USDT pair is taxed but users can launch a MedX-USDC pair to do the same trades
without taxes. Since the addresses are added manually by the owner or the admin with
the updateTaxedAddresses function any trades done before the pair is taxed will cause
the protocol to lose funds and the longer it takes to update the taxed addresses the
more funds the protocol will lose.

## Impact
The protocol will lose out on taxes they intend to get from DEX trades.

## Recommendation
Implement a system that dynamically checks if the to or from address in the _update
function is a DEX pair. This can be done by adding a new isDEXPair function that can
look for common patterns in DEX pairs. Do keep in mind that this solution will be gas
expensive.

Additionally, it would be better to have an off-chain bot that constantly tracks for any
pairs in any DEX, including MedX, and adds it to the taxed list automatically. Keep in
mind that these bots would need to have admin or owner privileges.

