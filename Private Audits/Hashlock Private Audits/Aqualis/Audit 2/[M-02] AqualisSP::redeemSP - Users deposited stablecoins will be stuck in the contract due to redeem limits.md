## Description
The `createSP` function allows users to deposit stablecoins and mint SP tokens depending
on the amount deposited minus the fees. Users can then use the `redeemSP` function to
burn their SP tokens, receiving any valid stablecoin. However, due to the minimum
redeem limit, user’s tokens will be stuck in the contract.

## Vulnerability Details
The `createSP` function shown below allows users to deposit any amount that is greater
than or equal to the `minimumStableInput` variable which is set to 20 USD worth of
tokens.
```solidity
function createSP(address _input, uint256 _amount) external watchTVL {
    if (!_stablecoins.contains(_input)) revert Errors.InvalidToken();

    uint256 inputDecimals = _decimals[_input];
    int128 amountFixed = _amount.divu(10 ** inputDecimals);
if (amountFixed < minimumStableInput) revert Errors.InsufficientAmount();
//rest of the function
```
However, due to the fees, if a user deposits for example 20e6 USDC, they will receive
less than 20e18 SP tokens.

When this user tries to withdraw their stablecoins with the redeemSP function shown
below:
```solidity
function redeemSP(address _output, uint256 _amount) external watchTVL {
    if (!_stablecoins.contains(_output)) revert Errors.InvalidToken();
    if (_amount < 20e18) revert Errors.InsufficientAmount();
//rest of the function
```
It will not be possible for this user to withdraw their stablecoins until they deposit more
stablecoins due to the user having less than 20e18 SP tokens.

## Proof of Concept
Implement the following test in the `LiquidityActions.t.sol` file and run it.
```solidity
function test_03_RedeemSPRevert() external {
    //users deposit
    vm.startPrank(USER_1);
    USDT.approve(address(SP), type(uint256).max);
    SP.createSP(address(USDT), 20 * 1e6);
    vm.stopPrank();
    vm.startPrank(USER_2);
    USDC.approve(address(SP), type(uint256).max);
    SP.createSP(address(USDC), 1000 * 1e6);
    vm.stopPrank();
    vm.startPrank(USER_3);
    DAI.approve(address(SP), type(uint256).max);
    SP.createSP(address(DAI), 1000 * 1e18);
    vm.stopPrank();
    uint256 USER_1SPbalance = SPToken.balanceOf(USER_1);
    uint256 USER_3SPbalance = SPToken.balanceOf(USER_3);
    //users redeem
    vm.prank(USER_1);
    vm.expectRevert();
    SP.redeemSP(address(USDT), USER_1SPbalance);
    vm.prank(USER_3);
    SP.redeemSP(address(DAI), USER_3SPbalance);
}
```
The test will revert as expected when USER_1 tries to redeem with all their SP tokens.

## Impact
User’s stablecoins will be stuck in the contract until they deposit more stablecoins to
mint more SP tokens.

## Recommendation
Change the minimum amount check in the createSP function as shown below to make
sure users can not have less than 20e18 SP tokens.
```solidity
function createSP(address _input, uint256 _amount) external watchTVL {
    if (!_stablecoins.contains(_input)) revert Errors.InvalidToken();
    uint256 inputDecimals = _decimals[_input];
    int128 amountFixed = _amount.divu(10 ** inputDecimals);
    (int128 fee, int128 newLiquidity) = getLiquidityFee(_input, _amount, false);
    /// @dev recursive depeg check
    if (isDepeg[_input]) {
        _depegCheck(_input, newLiquidity.div(_syncStablecoinBalances()));
        if (isDepeg[_input]) revert Errors.DepegProtection();
    }
    uint256 mintAmount = amountFixed.sub(fee).mul(getSharePrice()).mulu(
        1 ether
    );
    if (mintAmount < 20e18) revert Errors.InsufficientAmount();
    IERC20(_input).safeTransferFrom(msg.sender, address(this), _amount);
    SPtoken.mint(msg.sender, mintAmount);
    _accumulateFees(fee);
    /// @dev sync stablecoin balances and run depeg check
    _depegCheck(_input, newLiquidity.div(_syncStablecoinBalances()));
    emit SPCreated(msg.sender, _input, _amount, mintAmount);
}
```

