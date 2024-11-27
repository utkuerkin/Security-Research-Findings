## Description
The `updateWhitelist` function allows the owner or the admin to add or remove
addresses from the whitelist. Being in the whitelist means these addresses can send or
receive tokens without any taxes. This function or the `_updateWhitelist` function has no
checks to make sure taxed addresses can not be whitelisted.

## Vulnerability Details
The `updateWhitelist` and the `_updateWhitelist` functions shown below update the
whitelist status of given addresses.
```solidity
function updateWhitelist(
    address[] calldata removeFromWhitelist,
    address[] calldata addToWhitelist
) external onlyOwnerOrAdmin {
    uint256 removeListLength = removeFromWhitelist.length;
    for (uint256 i = 0; i < removeListLength; i++)
        _updateWhitelist(removeFromWhitelist[i], false);
    uint256 addListLength = addToWhitelist.length;
    for (uint256 i = 0; i < addListLength; i++)
        _updateWhitelist(addToWhitelist[i], true);
}
function _updateWhitelist(address account, bool whitelist) private {
    if (whitelisted[account] == whitelist) return;
    whitelisted[account] = whitelist;
    emit WhitelistUpdated(account, whitelist);
}
```
As observed in these functions, there are no checks implemented to make sure that the
address being whitelisted is not a taxed address. Taking a look at the `_update` function
below.
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
It is observed that a potential mistake whitelisting a taxed address (such as UniSwap
pairs) would lead to these addresses avoiding tax.

## Impact
Addresses that should be taxed on token transfers such as DEX pairs would not be
taxed if they were accidentally added to the whitelist.

## Recommendation
Implement a check in your functions to prevent taxed addresses from being added to
the whitelist. An example is shown below.
```solidity
function _updateWhitelist(address account, bool whitelist) private {
    if (applyTax[account] == true && whitelist == true) revert InvalidInputs();
    if (whitelisted[account] == whitelist) return;
    whitelisted[account] = whitelist;
    emit WhitelistUpdated(account, whitelist);
}
```

