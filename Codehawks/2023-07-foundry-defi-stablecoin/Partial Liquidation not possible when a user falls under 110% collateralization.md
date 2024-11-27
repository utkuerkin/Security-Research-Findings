### Severity

High Risk

### Relevant GitHub Links

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol

## **Summary**

When a `badUser` falls below 110% collateralization (For 
example: `badUser`’s collateral is $1099 worth of WETH and `badUser`’s 
`dscMinted` is 1000 DSC tokens.) It is not possible to partially liquidate
 them.

## **Vulnerability Details**

The current liquidation process uses a health factor improved check with

```jsx
uint256 endingUserHealthFactor = _healthFactor(user);
        if (endingUserHealthFactor <= startingUserHealthFactor) {
            revert DSCEngine__HealthFactorNotImproved();

```

but when a user falls under 110% collateralization it is 
not possible to improve their health factor with partial liquidation 
because of `totalCollateralToRedeem` being subtracted from `s_collateralDeposited[from][tokenCollateralAddress]`
 when liquidate function calls `redeemCollateral`. In an example case 
where the `badUser` has $1099 worth of WETH and 1000 DSC their 
`startingUserHealthFactor` would be 5495 * 1e14 (Fails the `healthCheck` 
making them eligible to being liquidated). When a liquidator for example
 tries to cover 500 DSC tokens of debt `_redeemCollateral` function will 
substract $550 worth of WETH from the `badUser` lowering their collateral 
to $549 worth of WETH, then `_burnDsc` function will burn 500 tokens and 
lower the `badUser`’s `s_DSCMinted` to 500 DSC tokens. This computation will
 make `endingUserHealthFactor` of `badUser` 5490 * 1e14. The liquidation 
function will revert due to not passing the if check at `if (endingUserHealthFactor <= startingUserHealthFactor)` even trying to cover 1 DSC token of debt would fail the if check.

Test function used is as follows:

```jsx
function testHundredTenPercentPartialLiquidation() public {
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
        amountToMint = 500 ether;
        dsce.liquidate(weth, user, amountToMint); // We are partially covering their debt
        vm.stopPrank();
    }

```

Foundry error is as follows: `[FAIL. Reason: DSCEngine__HealthFactorNotImproved()]`

## **Impact**

The impact of this vulnerability is that it stops 
potential liquidators from being able to partially liquidate a `badUser` 
that has fallen under 110% collateralization. Therefor impacting the 
stability of DSC token.

## **Tools Used**

Foundry and manual review.

## **Recommended Mitigation**

Implement an if check that ignores if the `healthFactor` of 
`badUser` is improved if they fall below 110% collateralization or 
implement an if check that allows only full liquidation of `badUser`'s that
 fall below 110% collateralization.

### Result
Validated,
Self-Duplicate