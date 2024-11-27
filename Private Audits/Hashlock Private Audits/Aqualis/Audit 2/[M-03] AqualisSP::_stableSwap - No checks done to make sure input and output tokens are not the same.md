## Description
The `_stableSwap` function allows users to swap their valid stablecoins for any valid
stablecoins. However, this function lacks checks to make sure that the input token is not
the same as the output token. Accidental user inputs will lead them to swapping the
same token and paying fees for no reason.

## Vulnerability Details
The `_stableSwap` function shown below allows users to swap stablecoins:
```solidity
function _stableSwap(
    address _input,
    address _output,
    uint256 _amountIn,
    uint256 _minAmountOut,
    address _to
) internal {
    uint256 inputDecimals = _decimals[_input];
    uint256 outputDecimals = _decimals[_output];
    int128 inputLiquidity = getLiquidity(_input, inputDecimals);
    int128 outputLiquidity = getLiquidity(_output, outputDecimals);
    int128 inputAmount = _amountIn.divu(10 ** inputDecimals);
    if (inputAmount < minimumStableInput) revert Errors.InsufficientAmount();
    int128 fee = _getStableSwapFee(
        inputLiquidity,
        outputLiquidity,
        inputAmount
    );
    uint256 amountOut = inputAmount.sub(fee).mulu(10 ** outputDecimals);
    if (_minAmountOut != 0 && amountOut < _minAmountOut)
        revert Errors.InsufficientSlippage();
    if (inputAmount.sub(fee) > outputLiquidity)
        revert Errors.InsufficientLiquidity(_output);
    IERC20(_input).safeTransferFrom(msg.sender, address(this), _amountIn);
    address to = _to == address(0) ? msg.sender : _to;
    IERC20(_output).safeTransfer(to, amountOut);
    _accumulateFees(fee);
    /// @dev Sync stablecoin balances after swap and run depeg check
    int128 totalStableBalance = _syncStablecoinBalances();
    _depegCheck(_input, inputLiquidity.div(totalStableBalance));
    _depegCheck(_output, outputLiquidity.div(totalStableBalance));
    emit Swap(_input, _output, _amountIn, amountOut, msg.sender, to);
}
```
This internal function is called by the `swapExactStableToStable` and
`swapStableToExactStable` functions shown below:
```solidity
function swapExactStableToStable(
    address _input,
    address _output,
    uint256 _amountIn,
    uint256 _minAmountOut,
    address _to
) external {
    if (!_stablecoins.contains(_input) || !_stablecoins.contains(_output)) {
        revert Errors.InvalidSwapPair();
    }
    _stableSwap(_input, _output, _amountIn, _minAmountOut, _to);
}
function swapStableToExactStable(
    address _input,
    address _output,
    uint256 _maxAmountIn,
    uint256 _amountOut,
    address _to
) external {
    uint256 amountIn = getAmountInStable(_input, _output, _amountOut);
    if (_maxAmountIn != type(uint256).max) {
        if (amountIn > _maxAmountIn) revert Errors.InsufficientSlippage();
    }
    _stableSwap(_input, _output, amountIn, 0, _to);
}
```
As seen in all the functions above, only parameter validation done is to make sure that
the inputted stablecoins are valid stablecoins accepted by the protocol. There are no
protections implemented to make sure that the input and output parameters are
different.

## Impact
Allowing the same token to be swapped with itself will lead to users paying fees for no
reason and losing out on tokens.

## Recommendation
Implement checks in the function to revert in the explained scenario. An example shown
below:
```solidity
function _stableSwap(
    address _input,
    address _output,
    uint256 _amountIn,
    uint256 _minAmountOut,
    address _to
) internal {
    if (_input == _output) revert Errors.InvalidSwapPair();
    //Rest of the function
}
```
