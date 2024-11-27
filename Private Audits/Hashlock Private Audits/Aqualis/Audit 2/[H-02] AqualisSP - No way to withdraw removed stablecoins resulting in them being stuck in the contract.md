## Description
The `removeStablecoinAddress` function allows the admin to remove an already added
stablecoin from the set of stablecoins that are allowed to be used in the contract. When
a stablecoin is removed it can no longer be used in the contracts functionality. After
removal, there are no ways to interact with these tokens and they will be stuck in the
contract.

## Vulnerability Details
The `removeStablecoinAddress` function shown below allows the admin to remove a
stablecoin:
```solidity
function removeStablecoinAddress(
    address _stablecoinAddress
) external onlyAdmin {
    if (_stablecoinAddress == address(0)) revert Errors.ZeroAddress();
    _removeStablecoin(_stablecoinAddress);
}
function _removeStablecoin(address _stablecoin) internal {
    if (_stablecoins.remove(_stablecoin)) {
        _decimals[_stablecoin] = 0x0;
        _recalculateTargetRatio();
        emit StablecoinRemoved(_stablecoin);
    }
}
```
When a stablecoin is removed, it will not be able to be used in `deposit`, `withdraw` or
`swap` functions.

## Impact
Stablecoin that is removed will not be intractable with or withdrawable from the
contract until it is added again resulting in these coins being stuck in the contract.

## Recommendation
Implement a new withdraw function that is only able to withdraw removed stablecoins
to a DAO or Admin address. However, be aware of the centralization risks that come
with existence of this function and take proper measures to make sure it can not be
used maliciously. The DAO or the Admin should supply the withdrawn stablecoin
amount in a valid stable coin with this function.

