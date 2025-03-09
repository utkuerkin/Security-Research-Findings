## Description  
The `getExceedsSizeLimit` function checks if a token transfer should fail. This is done by checking if the receiver address will or will not exceed the maximum wallet size if the inputted amount is added to their existing balance. However, this function also checks if the sender or the receiver is exempt from limits and if any are exempt, moves on with the transfer, which makes these checks insufficient.  

## Vulnerability Details  
The `getExceedsSizeLimit` function checks if a token transfer should fail:  

```solidity
function getExceedsSizeLimits(address from, address to, uint256 amount) internal view returns (bool status) {
    if (isLimitExempt[from] || isLimitExempt[to]) {
        return false;
    }
    if (amount > maximumTransactionSize || (balanceOf(to) + amount) > maximumWalletSize) {
        return true;
    }
}
```

As observed from the highlighted part, if the sender or the receiver is exempt from limits, the transaction will not revert due to size limits. However, this is problematic as the sender can be a limit-exempt address such as the router, owner, or the liquidity pool while the receiver is not limit-exempt. In this situation, the receiver address can exceed the maximum wallet size limit, breaking this invariant.  

## Impact  
Any user can exceed wallet size limits.  

## Recommendation  
Fix the function logic in order to sufficiently check for wallet size limits.

