## Description  
The `refundByOwner` function allows the owner to refund users who have deposited in the contract. This function implements a check to make sure that the inputted address is a supported token. In edge cases where users have deposited with a token but this token is removed after the `removeSupportedToken` function, these users will not be able to be refunded.

## Vulnerability Details 
The `refundByOwner` function allows the owner to refund users who have deposited in the contract:

```solidity
function refundByOwner(address _token, address _user, uint256 _amount) onlyOwner external {
    require(supportedTokens[_token], "token is not supported");
    require(userBalance[_user][_token] >= _amount, "balance is not enough");
    userBalance[_user][_token] -= _amount;
    if (userBalance[_user][_token] == 0) {
        userBought[_user] = false;
    }
    SafeERC20.safeTransfer(IERC20(_token), _user, _amount);
    emit Refund(_token, _user, _amount);
}
```

As observed in this function, the highlighted check makes sure that the token inputted is a supported token. However, this check is unnecessary as the following line already checks if the user has deposited using this token.

Taking a look at the `removeSupportedToken` function:

```solidity
function removeSupportedToken(address tokenAddress) onlyOwner external {
    require(supportedTokens[tokenAddress], "token not exist");
    supportedTokens[tokenAddress] = false;
    for (uint i = 0; i < supportedTokenList.length; i++) {
        if (supportedTokenList[i] == tokenAddress) {
            supportedTokenList[i] = supportedTokenList[supportedTokenList.length - 1];
            supportedTokenList.pop();
            break;
        }
    }
    emit RemoveTokenEvent(tokenAddress);
}
```

It is observed that, without any additional checks to see if users have deposited with this function, it will remove the supported token. When a supported token is removed this way, users will no longer be able to be refunded by the owner.

## Impact
Users will not be able to be refunded by the owner if the token they deposited with has been removed from the supported tokens.

## Recommendation  
Remove the check in the function that verifies if the token inputted is a supported token.

