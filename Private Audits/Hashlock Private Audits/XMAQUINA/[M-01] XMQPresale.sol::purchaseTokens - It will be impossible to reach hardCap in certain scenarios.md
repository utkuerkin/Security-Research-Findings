## Description
The `purchaseTokens` function allows users to purchase tokens in the presale. The purchase is only possible when the sold amount will not reach the `hardCap`, the amount to purchase is greater than the `minPurchaseAmount`, and the user purchase will not pass the `maxPurchaseAmount` threshold. However, there can be scenarios where it will be impossible to reach the `hardCap`. This will make it so the full intended amount of tokens cannot be sold.

## Vulnerability Details
The `purchaseTokens` function allows users to purchase tokens in the presale as long as some requirements are met.

```solidity
function purchaseTokens(uint256 amount) external payable nonReentrant whenNotPaused {
    if (block.timestamp < startTime || block.timestamp > dueTime) revert Errors.InvalidTime("Presale is not active");
    if (whitelistNFT.balanceOf(msg.sender) == 0) revert Errors.NotWhitelisted();
    if (totalTokensSold + amount > hardCap) revert Errors.ExceedsHardCap();
    if (amount < minPurchaseAmount) revert Errors.BelowMinimumPurchase();
    if (userPurchases[msg.sender] + amount > maxPurchaseAmount) revert Errors.ExceedsMaxPurchase();
    //rest of the function
}
```

Imagine the following scenario where `hardCap` is `1000` tokens, `minPurchaseAmount` is `50` tokens, and so far `951` tokens have been sold. The remaining `49` tokens to reach the `hardCap` will not be possible as any calls will either fail with the `ExceedsHardCap` error or the `BelowMinimumPurchase` error.

Take a look at the `canClaim` function below:

```solidity
function canClaim() public view returns (bool) {
    return (softCapReached && (totalTokensSold >= hardCap || block.timestamp > dueTime));
}
```

It is observed that even if a presale is successful and it should reach the `hardCap` with enough user interest in the presale, since it will not be possible to reach it, all users will have to wait until `dueTime` is reached to claim their tokens.

## Impact
It will be impossible to reach the `hardCap`, even if there’s enough interest in wanting to purchase tokens. Some amount of tokens will be unpurchasable. Users will need to wait until `dueTime` is reached to claim their tokens in all situations. The unpurchasable amount of tokens will be stuck in the contract.

## Recommendation
Update the function so that the full amount of tokens can be sold. If the `hardCap - totalTokensSold` amount is less than the `minPurchaseAmount`, allow the purchase of these remaining tokens.

