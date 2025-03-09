## Description
The `launch` function allows users to launch a new token and buy this token with the asset tokens they supply. When a token’s reserves reaches a predetermined ratio, these tokens become ready for graduation and migration into UniSwap pairs. However, in cases where the initial purchase during token’s deployment results in this ratio being reached, tokens are not graduated and require another purchase to be graduated.

## Vulnerability Details
The `launch` function allows users to launch a new token and buy this token with the asset tokens they supply.

```solidity
function launch(
    string memory name_,
    string memory ticker_,
    string memory prompt_,
    string memory intention_,
    string memory url_,
    uint256 purchaseAmount_
) external nonReentrant returns (address token, address pair, uint256 index) {
    // rest of the function

    IERC20(assetToken_).forceApprove(address(router), initialPurchase);
    router.buy(initialPurchase, address(actualToken), address(this));
    actualToken.transfer(msg.sender, actualToken.balanceOf(address(this)));

    return (address(actualToken), newPair, tokenIndex);
}
```

As observed in this function, it makes a purchase through the `router#buy` function instead of the `Manager#buy` function. Taking a look at the differences between these two functions:

### Manager.sol

```solidity
function buy(
    uint256 amountIn_,
    address tokenAddress_
) external payable returns (bool) {
    require(agentTokens[tokenAddress_].isTrading, "Trading not active");

    address pairAddress = factory.getPair(
        tokenAddress_,
        router.assetToken()
    );

    IPair pair = IPair(pairAddress);
    (uint256 reserveA, uint256 reserveB) = pair.getReserves();

    (uint256 amount1In, uint256 amount0Out) = router.buy(
        amountIn_,
        tokenAddress_,
        msg.sender
    );

    uint256 newReserveA = reserveA - amount0Out;
    uint256 newReserveB = reserveB + amount1In;

    _updateMetrics(
        tokenAddress_,
        newReserveA,
        newReserveB,
        amount1In
    );

    uint256 totalSupply = IERC20(tokenAddress_).totalSupply();
    require(totalSupply > 0, "Invalid total supply");

    uint256 reservePercentage = (newReserveA * 100) / totalSupply;
    
    if (reservePercentage <= gradThresholdPercent) {
        _graduate(tokenAddress_);
    }

    return true;
}
```

### Router.sol

```solidity
function buy(
    uint256 amountIn_,
    address tokenAddress_,
    address to_
) external onlyRole(EXECUTOR_ROLE) nonReentrant returns (uint256, uint256) {
    require(tokenAddress_ != address(0), "Invalid token");
    require(to_ != address(0), "Invalid recipient");
    require(amountIn_ > 0, "Invalid amount");
    
    // Check token hasn't graduated
    Token token = Token(tokenAddress_);
    require(!token.hasGraduated(), "Token graduated");

    address pair = factory.getPair(tokenAddress_, assetToken);

    // Calculate split fees using Factory's tax settings
    uint256 feePercent = factory.buyTax();
    uint256 totalFee = (feePercent * amountIn_) / 10000;
    uint256 halfFee = totalFee / 2;
    uint256 finalAmount = amountIn_ - totalFee;
    
    address taxVault = factory.taxVault();
    address tokenOwner = token.owner();

    // Transfer tokens with split fees
    IERC20(assetToken).safeTransferFrom(to_, pair, finalAmount);
    IERC20(assetToken).safeTransferFrom(to_, taxVault, halfFee);
    IERC20(assetToken).safeTransferFrom(to_, tokenOwner, halfFee);

    uint256 amountOut = _getAmountsOut(tokenAddress_, assetToken, finalAmount);

    IPair(pair).transferTo(to_, amountOut);
    IPair(pair).swap(0, amountOut, finalAmount, 0);

    return (finalAmount, amountOut);
}
```

It is observed that the `Manager#buy` function checks if this purchase makes the token pass the graduation threshold while the `Router#buy` function does not have such checks. This means that even if the initial purchase is enough to have a token graduated, it will not be graduated and it will require another purchase of this token to be made. This results in inconsistent protocol behaviour, tokens that should be graduated not graduating and could affect trading strategies expecting graduation.

## Proof of Concept
Implement the following test in the `Manager.test.ts` file and run it to observe the vulnerability through terminal outputs.

```javascript
describe("Launch Graduation Check", function() {
    it.only("should not allow graduation from initial purchase", async function() {
        const { manager, router, alice, bob, assetToken } = context;
        
        console.log("\n=== Testing Launch Graduation Threshold ===");

        // 1. Alice launches token with large initial purchase
        const launchAmount = ethers.parseEther("20000"); // Large initial purchase
        await assetToken.connect(alice).approve(await manager.getAddress(), launchAmount);
        
        const tx = await manager.connect(alice).launch(
            "Test Token",
            "TEST",
            "Test prompt",
            "Test intention",
            "https://test.com",
            launchAmount
        );
        
        const receipt = await tx.wait();
        const event = receipt.logs.find(log => log.fragment?.name === "Launched");
        const agentToken = await ethers.getContractAt("Token", event.args.token);
        
        // Check graduation status after launch
        const tokenDataAfterLaunch = await manager.agentTokens(await agentToken.getAddress());
        console.log("\nAfter Launch:");
        console.log("Has Graduated:", tokenDataAfterLaunch.hasGraduated);
        console.log("Is Trading:", tokenDataAfterLaunch.isTrading);
        
        // 2. Bob makes tiny purchase
        const buyAmount = ethers.parseEther("0.0001");
        await assetToken.connect(bob).approve(await router.getAddress(), buyAmount);
        await manager.connect(bob).buy(buyAmount, await agentToken.getAddress());
        
        // Check graduation status after small buy
        const tokenDataAfterBuy = await manager.agentTokens(await agentToken.getAddress());
        console.log("\nAfter Small Buy:");
        console.log("Has Graduated:", tokenDataAfterBuy.hasGraduated);
        console.log("Is Trading:", tokenDataAfterBuy.isTrading);
        
        // Verify states
        expect(tokenDataAfterLaunch.hasGraduated).to.be.false;
        expect(tokenDataAfterBuy.hasGraduated).to.be.true;
        
        console.log("\n=== Launch Graduation Test Complete ===");
    });
});
```

## Impact
This vulnerability results in token graduation not happening when it should, results in inconsistent protocol behaviour, and could affect trading strategies revolving around token graduation.

## Recommendation
Implement the graduation check after the `buy` call made in the launch function. Alternatively, implement checks in the launch function where the initial purchase can not exceed an amount that would trigger graduation.
