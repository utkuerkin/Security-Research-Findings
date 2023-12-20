# 

### Severity

High Risk

### Relevant GitHub Links

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol

# Liquidation 110% problem

## **Summary**

This vulnerability arises when we try to liquidate a 
badUser that has fallen below 110% collateralization(For example: 
badUser’s collateral is $1099 worth of WETH and badUsers dscMinted is 
1000 tokens.) where they are still profitable for a liquidator. This 
issue causes the  s_collateralDeposited[from][tokenCollateralAddress] -=
 amountCollateral computation in _redeemCollateral function to result in
 an underflow. As a result, liquidation of the badUser’s position would 
not be possible, posing a risk to the stability and health of the 
system.

## **Vulnerability Details**

When the liquidator calls the liquidate function on a 
badUser that has fallen below 110% collateralization for example: 
badUser’s collateral is $1099 worth of WETH and badUsers dscMinted is 
1000 tokens (which still makes it profitable for liquidator since it is 
over 100% collateralization.) the liquidate function will call 
_redeemCollateral function with the totalCollateralToRedeem parameter 
that is computed with tokenAmountFromDebtCovered + bonusCollateral which
 will be equal to $1100 worth of WETH. This would cause `s_collateralDeposited[from][tokenCollateralAddress] -= amountCollateral` to revert with an underflow error since `s_collateralDeposited[from][tokenCollateralAddress]`
 of badUser is $1099 worth of WETH, making it impossible to liquidate a 
badUser that is still profitable to liquidate for the liquidator.

Test used for this vulnerability is as follows:

```jsx
function testHundredTenPercentLiquidate() public {
        vm.startPrank(user);
        amountCollateral = 1 ether;
        amountToMint = 1000 ether;
        ERC20Mock(weth).approve(address(dsce), amountCollateral);
        dsce.depositCollateralAndMintDsc(weth, amountCollateral, amountToMint);
        vm.stopPrank();
        int256 ethUsdUpdatedPrice = 1099e8; // 1 ETH = $1099

        MockV3Aggregator(ethUsdPriceFeed).updateAnswer(ethUsdUpdatedPrice);
        ERC20Mock(weth).mint(liquidator, collateralToCover);

        vm.startPrank(liquidator);
        collateralToCover = 20 ether;
        amountToMint = 2000 ether;
        ERC20Mock(weth).approve(address(dsce), collateralToCover);
        dsce.depositCollateralAndMintDsc(weth, collateralToCover, amountToMint);
        dsc.approve(address(dsce), amountToMint);
        amountToMint = 1000 ether;
        dsce.liquidate(weth, user, amountToMint); // We are covering their whole debt
        vm.stopPrank();
    }

```

This will cause the following error: `[FAIL. Reason: Arithmetic over/underflow]`

## **Impact**

When token prices decline significantly, this 
vulnerability might prevent the system from protecting itself against 
defaulting users since it is not possible to liquidate , increasing the 
risk of financial losses and potential instability. The normal 
liquidation limit would be when the token is 100% collateralized since 
under 100% would not make liquidation profitable for the liquidator but 
due to this issue users that go under 110% collateralization can not be 
liquidated and the system would lose stability since bad users will not 
get liquidated.

## **Tools Used**

Foundry and manual reviewing.

## **Recommended Mitigation**

There should be an if statement implemented to the 
liquidate function that will redeem all collateral the bad user has to 
the person that liquidates them instead of the 10% bonus in case their 
collateral value goes below totalCollateralToRedeem.

### Result
Validated