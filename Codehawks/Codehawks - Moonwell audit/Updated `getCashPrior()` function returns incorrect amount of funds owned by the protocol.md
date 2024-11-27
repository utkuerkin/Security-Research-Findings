# Moonwell - Findings Report


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Moonwell

### Dates: Mar 4th, 2024 - Mar 11th, 2024

[See more contest details here](https://www.codehawks.com/contests/clt7ewpli0001w7f6ol2yojki)




# Low Risk Findings

## <a id='L-01'></a>L-01. Updated `getCashPrior()` function returns incorrect amount of funds owned by the protocol.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-03-Moonwell/blob/e57b8551a92824d35d4490f5e7f27c373be172bd/src/MErc20DelegateFixer.sol#L129-L135

## Summary

Updated `getCashPrior()` function returns incorrect amount of funds owned by the protocol.

## Vulnerability Details

`getCashPrior()` function is changed in order to not have an impact on the exchange rate with the update. 
Current version:

```
function getCashPrior() internal view returns (uint256) {
        return EIP20Interface(underlying).balanceOf(address(this));
    }
```

Updated version

```
function getCashPrior() internal view returns (uint256) {
        return EIP20Interface(underlying).balanceOf(address(this)) + badDebt;
    }
```

This causes functions relying solely on `getCashPrior()` without using `totalBorrows` will receive incorrect data, leading to those functions not reverting when they should. Namely these 3 functions are: `borrowFresh(), _reduceReservesFresh() and redeemFresh()`

These functions have the following if check to see if the protocol has enough funds to support the transaction.

```
        if (getCashPrior() < vars.redeemAmount) {
            return fail(Error.TOKEN_INSUFFICIENT_CASH, FailureInfo.REDEEM_TRANSFER_OUT_NOT_POSSIBLE);
        }
```

However after `fixUser()`calls, in an example scenario where a user calls `redeemFresh()` with amount of tokens that exceeds the tokens protocol has, updated `getCashPrior()` will return a uint bigger than the amount of tokens the protocol has. This means that the if check here will be bypassed when it should be returning the error message provided in this function. In this scenario the function would revert at the following external call and receive the wrong error message.

```jsx
doTransferOut(redeemer, vars.redeemAmount);
```

## Impact

This vulnerability causes user’s to get the wrong error message on why their transaction has failed, shows the wrong amount of tokens owned by the protocol until the `badDebt` is paid. This can cause bigger issues on the front-end systems. Which is why this is a low impact vulnerability.

## Tools Used

Manual review.

## Recommendations

`borrowFresh(), _reduceReservesFresh() and redeemFresh()` should be updated to read 

```jsx
EIP20Interface(underlying).balanceOf(address(this))
```

instead of `getCashPrior()` function.

A better solution would be implementing a new function to be used in exchange rate maths that includes `badDebt` and leaving `getCashPrior()` as it is.


