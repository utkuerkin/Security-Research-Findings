## Description
The `createSP` function allows users to deposit stablecoins and mint `SP` tokens depending
on the amount deposited minus the fees. Users can then use the `redeemSP` function to
burn their `SP` tokens, receiving any valid stablecoin. However, when the stablecoin pools
are unbalanced the calculation breaks, leading to users losing funds.

## Vulnerability Details
The `redeemSP` function shown below allows users to burn their `SP` tokens and receive
any valid stablecoin in return.
```solidity
function redeemSP(address _output, uint256 _amount) external watchTVL {
    if (!_stablecoins.contains(_output)) revert Errors.InvalidToken();
    if (_amount < 20e18) revert Errors.InsufficientAmount();
    (int128 fee, int128 newLiquidity) = getLiquidityFee(_output, _amount, true);
    int128 outputAmount = _amount.divu(1e18).sub(fee).mul(getSharePrice());
    uint256 transferAmount = outputAmount.mulu(10 ** _decimals[_output]);
    uint256 availableLiquidity = IERC20(_output).balanceOf(address(this));
    if (transferAmount > availableLiquidity)
        revert Errors.InsufficientLiquidity(_output);
    SPtoken.burn(msg.sender, _amount);
    IERC20(_output).safeTransfer(msg.sender, transferAmount);
    _accumulateFees(fee);
    /// @dev sync stablecoin balances and run depeg check
    _depegCheck(_output, newLiquidity.div(_syncStablecoinBalances()));
    emit SPRedeemed(msg.sender, _output, transferAmount, _amount);
}
```
This function will calculate the `fee` and `newLiquidity` in the `getLiquidityFee` function
shown below:
```solidity
function getLiquidityFee(
    address _stablecoin,
    uint256 _amount,
    bool _isExit
) public view returns (int128 fee, int128 newLiquidityFixed) {
    uint256 decimals = _decimals[_stablecoin];
    uint256 amountDecimals = _isExit ? 18 : decimals;
    int128 amountFixed = _amount.divu(10 ** amountDecimals);
    uint256 initialLiquidity = IERC20(_stablecoin).balanceOf(address(this));
    if (initialLiquidity == 0)
        return (amountFixed.mul(liquidityBaseFee), amountFixed);
    int128 initialLiquidityFixed = initialLiquidity.divu(10 ** decimals);
    int128 totalLiquidityFixed = totalStablecoinBalance.add(
        totalSuppliedToLending
    );
    int128 newTotalLiquidityFixed;
    if (!_isExit) {
        newLiquidityFixed = initialLiquidityFixed.add(amountFixed);
        newTotalLiquidityFixed = totalLiquidityFixed.add(amountFixed);
    } else {
        newLiquidityFixed = initialLiquidityFixed.sub(amountFixed);
        newTotalLiquidityFixed = totalLiquidityFixed.sub(amountFixed);
    }
    int128 initialRatioFixed = initialLiquidityFixed.div(totalLiquidityFixed);
    int128 newRatioFixed = newLiquidityFixed.div(newTotalLiquidityFixed);
    int128 initialDiff = ABDKMath64x64.abs(
        liquidityTargetRatio.sub(initialRatioFixed)
    );
    int128 newDiff = ABDKMath64x64.abs(newRatioFixed.sub(liquidityTargetRatio));
    int128 avgDiff = ABDKMath64x64.avg(initialDiff, newDiff);
    int128 taxOrRebate = liquidityBaseTax.mul(avgDiff).div(
        liquidityTargetRatio
    );
    if (newDiff < initialDiff) {
        // This action benefits the Liquidity ratio, thus getting rebate instead of tax
        fee = amountFixed.mul(liquidityBaseFee.sub(taxOrRebate));
    } else {
        // This action reduce the Liquidity ratio, thus getting taxed
        fee = amountFixed.mul(liquidityBaseFee.add(taxOrRebate));
    }
}
```
When the pools are nearly drained, this function will return a really high `fee` and a
negative `newLiquidity` amount. This will result in users getting less stablecoins than they
should.

## Proof of Concept
Implement the following test in the `LiquidityActions.t.sol` file and run it to observe the
token loss.
```solidity
function test_03_RedeemSPIssue() external {
    uint256 USER_2USDTbalanceBefore = USDT.balanceOf(USER_2);
    //users deposit
    vm.startPrank(USER_1);
    USDT.approve(address(SP), type(uint256).max);
    SP.createSP(address(USDT), 30 * 1e6);
    vm.stopPrank();
    vm.startPrank(USER_3);
    DAI.approve(address(SP), type(uint256).max);
    SP.createSP(address(DAI), 1000 * 1e18);
    vm.stopPrank();
    vm.startPrank(USER_4);
    USDC.approve(address(SP), type(uint256).max);
    SP.createSP(address(USDC), 1000 * 1e6);
    vm.stopPrank();
    vm.startPrank(USER_2);
    USDT.approve(address(SP), type(uint256).max);
    SP.createSP(address(USDT), 1800 * 1e6);
    vm.stopPrank();
    uint256 USER_1SPbalance = SPToken.balanceOf(USER_1);
    uint256 USER_2SPbalance = SPToken.balanceOf(USER_2);
    uint256 USER_3SPbalance = SPToken.balanceOf(USER_3);
    uint256 USER_4SPbalance = SPToken.balanceOf(USER_4);
    //users redeem
    vm.prank(USER_1);
    SP.redeemSP(address(USDT), USER_1SPbalance);
    vm.prank(USER_3);
    SP.redeemSP(address(USDT), USER_3SPbalance);
    vm.prank(USER_4);
    SP.redeemSP(address(DAI), USER_4SPbalance);
    vm.prank(USER_2);
    SP.redeemSP(address(USDT), USER_2SPbalance);
    //logs
    uint256 USER_2USDTbalanceAfter = USDT.balanceOf(USER_2);
    uint256 USER_2USDTLostAmount = USER_2USDTbalanceBefore -
        USER_2USDTbalanceAfter;
    console2.log("USER2 USDT lost:", USER_2USDTLostAmount);
    console2.log("USER2 SP balance after", SPToken.balanceOf(USER_2));
}
```
This test will print the following logs:
```javascript
USER2 USDT lost: 1554151524
USER2 SP balance after 0
```
Showing that USER_2 lost most of their USDT while burning all of their SP.
1800e18 SP burned and received 245.8e6 USDT in return

## Impact
Users will receive less than the intended amount when they call the redeemSP function.

## Recommendation
Implement a new parameter for users to enter their minimum desired stable coin
amount and revert if this condition is not met. An example shown below:
```solidity
function redeemSP( address _output, uint256 _amount, uint256 minAmountOut) external watchTVL {
    //rest of the function
    uint256 transferAmount = outputAmount.mulu(10 ** _decimals[_output]);
    if (transferAmount < minAmountOut) revert Errors.InsufficientAmount();
    //rest of the function
}
```
