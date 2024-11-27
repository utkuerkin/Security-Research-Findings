## Description
The `removeStablecoinAddress` function allows the admin to remove an already added
stablecoin from the set of stablecoins that are allowed to be used in the contract. When
a stablecoin is removed it can no longer be used in the contracts functionality.

The `redeemSP` function allows users to withdraw their choice of stablecoins while
burning their SP tokens. However, when a stablecoin is removed, the user that redeems
rst will get the stablecoin amount that they should. This rst redeem will sync
stablecoin balances and subsequent users will receive less stablecoins when they
redeem.

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

The `redeemSP` function shown below allows users to withdraw stablecoins:
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
When a stablecoin is removed and the stablecoin balances in the contract are synced
due to the current imbalance in minted SP and stablecoin balances, the worth of an SP
token will be lowered and users will be receiving less stablecoins than they should when they redeem.

If a user redeems before the sync of balances, when they redeem they will get the
stablecoin amount they should and this will sync the stablecoin balances in the contract.
Subsequent redeems will cause users to receive less stablecoins than they should.

## Proof of Concept
Implement the following test in LiquidityActions.t.sol le, run it and observe the logs to
notice the loss of tokens
```solidity
function test_04_RedeemSPAfterRemoval() external {
    uint256 USER_1USDTBefore = USDT.balanceOf(USER_1);
    uint256 USER_3USDCBefore = USDC.balanceOf(USER_3);
    //users deposit
    vm.startPrank(USER_1);
    USDT.approve(address(SP), type(uint256).max);
    SP.createSP(address(USDT), 1000 * 1e6);
    vm.stopPrank();
    vm.startPrank(USER_3);
    USDC.approve(address(SP), type(uint256).max);
    SP.createSP(address(USDC), 1000 * 1e6);
    vm.stopPrank();
    vm.startPrank(USER_4);
    DAI.approve(address(SP), type(uint256).max);
    SP.createSP(address(DAI), 1000 * 1e18);
    vm.stopPrank();
    //admin removes DAI
    vm.startPrank(ADMIN);
    SP.removeStablecoinAddress(address(DAI));
    vm.stopPrank();
    uint256 USER_1SPBalance = SPToken.balanceOf(USER_1);
    uint256 USER_3SPBalance = SPToken.balanceOf(USER_3);
    //users redeem
    vm.startPrank(USER_1);
    SP.redeemSP(address(USDT), USER_1SPBalance);
    vm.stopPrank();
    vm.startPrank(USER_3);
    SP.redeemSP(address(USDC), USER_3SPBalance);
    vm.stopPrank();
    uint256 USER_1USDTAfter = USDT.balanceOf(USER_1);
    uint256 USER_3USDCAfter = USDC.balanceOf(USER_3);
    console2.log("User1 USDT lost:", USER_1USDTBefore - USER_1USDTAfter);
    console2.log("User3 USDC lost:", USER_3USDCBefore - USER_3USDCAfter);
}
```
The test will print out the following logs:
```javascript
User1 USDT lost: 458273
User3 USDC lost: 499886974
```
As seen from the test logs, USER_1 received the stablecoin amount that they should
since the balances are not synced. USER_3 receives less stablecoin since the balances
are synced at this point.

## Impact
When a stablecoin is removed and this is not compensated in any way by adding a new
stablecoin with the same amount, users will end up receiving less stablecoins than they
should. The rst user to redeem will receive the amount that they should and
subsequent users will receive less tokens due to the stablecoin balances in the contract
being synced after the first redeem.

## Recommendation
When a stablecoin is removed the DAO or the Admin should supply the removed
stablecoin’s amount in a valid stable coin to keep contract functionality working as
intended. Removal and addition should be done in one transaction and ideally in a single
function.

