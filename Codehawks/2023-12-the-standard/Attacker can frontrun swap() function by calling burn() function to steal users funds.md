# The Standard - Findings Report


### Sponsor: The Standard

### Dates: Dec 27th, 2023 - Jan 10th, 2024

[See more contest details here](https://www.codehawks.com/contests/clql6lvyu0001mnje1xpqcuvl)



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Attacker can frontrun `swap()` function by calling `burn()` function to steal users funds.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L206-L235

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L169-L175

## Summary

User can lose funds when calling `swap()` function in `SmartVaultV3.sol` , due to how calculations in `calculateMinimumAmountOut()` sets `minimumAmountOut` in `swap()`. An attacker can manipulate a users slippage during a swap.

## Vulnerability Details

`swap()` [function](https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L214-L235) allows a user to swap their collateral with a different token. When a user calls `swap()` , this function makes an internal call to `calculateMinimumAmountOut()` [function](https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L206-L212) to set `minimumAmountOut` and make it so user won’t be in a position to be liquidated.

```jsx
function calculateMinimumAmountOut(bytes32 _inTokenSymbol, bytes32 _outTokenSymbol, uint256 _amount) private view returns (uint256) {
        ISmartVaultManagerV3 _manager = ISmartVaultManagerV3(manager);
        uint256 requiredCollateralValue = minted * _manager.collateralRate() / _manager.HUNDRED_PC();
        uint256 collateralValueMinusSwapValue = euroCollateral() - calculator.tokenToEur(getToken(_inTokenSymbol), _amount);
        return collateralValueMinusSwapValue >= requiredCollateralValue ?
            0 : calculator.eurToToken(getToken(_outTokenSymbol), requiredCollateralValue - collateralValueMinusSwapValue);
    }
```

For example a normal use case for this function can go like shown below:
1) User has minted 100 EUROs by depositing 150€ worth of wETH.

2) User notices ETH will lose value so they call `swap()` function with `_amount` set to 150 to swap their wETH to USDC in order to not be under collateralized.

3) `swap()` function does a call to `calculateMinimumAmountOut()` 

4) `calculateMinimumAmountOut()` returns 120€ worth of tokens. Math for this calculation when `collateralRate()` is set to 120 like in unit tests goes as shown below:

- `requiredCollateralValue` = $100 * 120 / 100 = 120$
- `collateralValueMinusSwapValue` = $150 - 150 = 0$

5) In this normal scenario user can get a minimum of 120€ worth of USDC from their swap because `minimumAmountOut` is set to 120€ worth of tokens.

In a potential attack scenario this same swap can go like shown below:

1) User has minted 100 EUROs by depositing 150€ worth of wETH.

2) User notices ETH will lose value so they call `swap()` function with `_amount` set to 150 to swap their wETH to USDC in order to not be under collateralized.

3) Attacker does a frontrunning attack and calls `burn()` function with `_amount` = 100 (`minted` becomes 0) and also performing a deposit into UniSwap pools to manipulate prices (first part of the sandwich attack).

4) `swap()` function does a call to `calculateMinimumAmountOut()` 

5) `calculateMinimumAmountOut()` returns 0€ worth of tokens. Math for this calculation when `collateralRate()` is set to 120 like in unit tests goes as shown below:

- `requiredCollateralValue` = $0 *120/100 = 0$
- `collateralValueMinusSwapValue` = $150-150 = 0$

6) With this front running attack `minimumAmountOut` is set to 0.

7) Attacker can now finish the sandwich attack in order to steal user’s funds and easily steal more than the money they have spent when calling `burn()`. Using the numbers given in the example, attacker is able to steal up to 150€ worth of tokens while spending 100 EUROs to call the `burn()` function. User risks losing all their collateral considering their `minted` is set to 0 by the attacker, user would end up with a loss of 50€ worth of funds. Since value of collateral is always more than the value of `minted`, user would always lose money.

Readme file of this protocol acknowledges a sandwich attack since the following line is in Known Issues section: 

> `minimumAmountOut` can be set to 0 if there is no value required to keep collateral above required level. The user is likely to lose some value in the swap, especially when Uniswap fees are factored in, but this is at the user's discretion
> 

However this attack is not due to a user error, this is an attacker causing `minimumAmountOut` to be set to 0.
Numbers I have used are just for simplicity sake to explain the attack. Attacker doesn’t need to `burn` all the `minted` amount to set `minimumAmountOut` to 0, attacker would simply need to `burn` just enough to set `collateralValueMinusSwapValue >= requiredCollateralValue` [line](https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L210C12-L210C12) to `true`.

## Impact

This vulnerability directly causes a users funds to be stolen.

## Tools Used

Manual review.

## Recommendations

A simple solution would be to add `onlyOwner` modifier to `burn()` function.

```jsx
function burn(uint256 _amount) external ifMinted(_amount) onlyOwner {
//rest of the function
} 
```

A better solution would be to add a new parameter for the user to set their minimum amount expected from the swap in `swap()` function and implementing an if check. Updated `swap()` function is shown below.

```jsx
function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount, uint256 _minimumAmountExpected) external onlyOwner {
        uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        address inToken = getSwapAddressFor(_inToken);
        uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
+       if (_minimumAmountExpected > minimumAmountOut){
+           minimumAmountOut = _minimumAmountExpected;
        }
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
                tokenIn: inToken,
                tokenOut: getSwapAddressFor(_outToken),
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: _amount - swapFee,
                amountOutMinimum: minimumAmountOut,
                sqrtPriceLimitX96: 0
            });
        inToken == ISmartVaultManagerV3(manager).weth() ?
            executeNativeSwapAndFee(params, swapFee) :
            executeERC20SwapAndFee(params, swapFee);
    }

    function setOwner(address _newOwner) external onlyVaultManager {
        owner = _newOwner;
    }
```

This allows users to have more protection with their swap and stops this potential frontrunning and sandwich attack. Consider implementing both solutions.


