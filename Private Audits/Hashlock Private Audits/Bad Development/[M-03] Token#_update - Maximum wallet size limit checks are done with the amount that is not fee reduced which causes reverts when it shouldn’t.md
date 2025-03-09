## Description
BAD token imposes a maximum wallet size limit on its users. Any transaction is checked if the receiving address exceeds this limit as a result of this transaction. However, the amount that is used to check if the user will exceed this limit is not the actual amount that the user will receive after fees. This causes the transactions to revert when they shouldn’t.

## Vulnerability Details
The `getExceedsSizeLimit` function is used in the `_update` function to see if the result of the transaction will cause the receiving address to exceed the maximum wallet size limit.
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
Take a look at the `_update` function.

```solidity
function _update(address from, address to, uint256 value) internal virtual override {
    // Rest of the function
} else {
    if (getExceedsSizeLimits(from, to, value)) {
        revert ExceedsSizeLimit();
    }

    if (shouldTakeFee(from, to)) {
        uint256 transferAmount = value;
        uint256 feeAmount = getFeeAmount(to, transferAmount);
        transferAmount -= feeAmount;

        super._update(from, address(this), feeAmount);
        // Rest of the function
        // Send on to caller
        super._update(from, to, transferAmount);
}
```

It is observed that the size limits are checked with the inputted value while the user wallet receives `value - feeAmount`. This means that even if the user does not exceed the wallet size after fees are subtracted from the transfer amount, the transaction will still fail. 

In an example scenario where the user is 99 tokens short of the maximum wallet size:

- The user gets transferred 100 tokens.
- With 5% fees, the user will receive 95 tokens.
- The transaction will still fail as the maximum wallet size check is done with the user balance + 100 instead of the actual amount the user receives.

## Impact
Transactions that should not fail will fail as the contract incorrectly calculates the user’s token amount after the transaction.

## Recommendation
Correctly check if the user exceeds the maximum wallet size by using the amount that is subtracted with fee amounts if fees are applicable to this transaction.
