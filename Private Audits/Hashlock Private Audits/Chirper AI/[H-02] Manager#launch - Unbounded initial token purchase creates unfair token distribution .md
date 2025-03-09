## Description
The `launch` function allows users to launch a new token and buy this token with the asset tokens they supply. Due to the calculations, early purchases are at an advantage where they receive more launched tokens for the same amount of asset token supplied. The initial purchase of the launched token gets the best rate and since there are no limitations on the amount, this creates a potential rug pull risk where a single account owns most of the tokens.

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
    address assetToken_ = router.assetToken();
    require(
        IERC20(assetToken_).balanceOf(msg.sender) >= purchaseAmount_,
        "Insufficient funds"
    );

    uint256 launchTax = (purchaseAmount_ * factory.launchTax()) / 10000;
    uint256 initialPurchase = purchaseAmount_ - launchTax;
    
    // Transfer launch tax to tax vault
    IERC20(assetToken_).safeTransferFrom(msg.sender, factory.taxVault(), launchTax);
    IERC20(assetToken_).safeTransferFrom(
        msg.sender,
        address(this),
        initialPurchase
    );

    // Create token with factory reference
    Token actualToken = new Token(
        string.concat(name_, "agent"), 
        ticker_,
        initialSupply,
        maxTxPercent,
        address(factory),
        address(this)
    );
    uint256 supply = actualToken.totalSupply();

    address newPair = factory.createPair(address(actualToken), assetToken_);

    require(_approve(address(router), address(actualToken), supply));

    uint256 k = ((K * 10000) / assetRate);
    uint256 liquidity = (((k * 10000 ether) / supply) * 1 ether) / 10000;

    router.addInitialLiquidity(address(actualToken), supply, liquidity);

    TokenMetrics memory metrics = TokenMetrics({
        token: address(actualToken),
        name: string.concat(name_, "agent"),
        baseName: name_,
        ticker: ticker_,
        supply: supply,
        price: supply / liquidity,
        mktCap: liquidity,
        liq: liquidity * 2,
        vol: 0,
        vol24h: 0,
        lastPrice: supply / liquidity,
        lastUpdate: block.timestamp
    });

    TokenData memory localToken = TokenData({
        creator: msg.sender,
        token: address(actualToken),
        pair: newPair,
        prompt: prompt_,
        intention: intention_,
        url: url_,
        metrics: metrics,
        isTrading: true,
        hasGraduated: false
    });

    agentTokens[address(actualToken)] = localToken;
    agentTokenList.push(address(actualToken));

    uint256 tokenIndex = agentTokenList.length;

    emit Launched(address(actualToken), newPair, tokenIndex);

    IERC20(assetToken_).forceApprove(address(router), initialPurchase);
    router.buy(initialPurchase, address(actualToken), address(this));
    actualToken.transfer(msg.sender, actualToken.balanceOf(address(this)));

    return (address(actualToken), newPair, tokenIndex);
}
```

As observed in this function, there are no limits on the percentage of the total supply bought at the initial purchase. Taking a look at the `Token#_update` function we can observe that subsequent purchases after the launch would be limited.

```solidity
function _update(
    address from_,
    address to_,
    uint256 amount_
) private {
    require(from_ != address(0), "Invalid sender");
    require(to_ != address(0), "Invalid recipient");
    require(amount_ > 0, "Invalid amount");

    // Check transaction limit if sender is not exempt
    if (!transactionLimitExempt[from_]) {
        require(
            amount_ <= maxTransactionAmount,
            "Transfer amount exceeds transaction limit"
        );
    }
}
```

This shows an inconsistency where the initial purchase is at a much greater advantage where they can buy an unlimited amount of tokens at the best rate while other users would have to buy at worse rates at limited amounts. 

All of the above taken into consideration, the launched tokens will have a risk where the token deployer will hold most of the total supply of this token which will lead to issues such as the deployer rug pulling other users due to the centralized distribution of tokens.

## Proof of Concept
Implement the following test in the `Manager.test.ts` file and run it to observe the vulnerability through the logs.

```javascript
describe("Token Launch Price Advantage", function() {
    it.only("demonstrates price advantage for token launcher", async function() {
        const { manager, router, alice, bob, assetToken } = context;
        
        console.log("\n=== Testing Launch Price Advantage ===");

        // 1. Alice launches token with 20000 USDC
        const launchAmount = ethers.parseEther("20000");
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
        
        // Get Alice's token balance and total supply
        const aliceBalance = await agentToken.balanceOf(alice.address);
        const totalSupply = await agentToken.totalSupply();
        const alicePercent = (Number(aliceBalance) * 100) / Number(totalSupply);
        
        console.log("\nLaunch Statistics:");
        console.log("Total Supply:", ethers.formatEther(totalSupply));
        console.log("Alice's USDC spent:", ethers.formatEther(launchAmount));
        console.log("Alice's tokens received:", ethers.formatEther(aliceBalance));
        console.log("Alice's % of total supply:", alicePercent.toFixed(2) + "%");
        console.log("Rate (tokens/USDC):", Number(aliceBalance) / Number(launchAmount));

        // 2. Bob buys with 20000 USDC
        const buyAmount = ethers.parseEther("20000");
        await assetToken.connect(bob).approve(await router.getAddress(), buyAmount);
        await manager.connect(bob).buy(buyAmount, await agentToken.getAddress());
        
        // Get Bob's token balance and percentage
        const bobBalance = await agentToken.balanceOf(bob.address);
        const bobPercent = (Number(bobBalance) * 100) / Number(totalSupply);
        
        console.log("\nFirst Buy Statistics:");
        console.log("Bob's USDC spent:", ethers.formatEther(buyAmount));
        console.log("Bob's tokens received:", ethers.formatEther(bobBalance));
        console.log("Bob's % of total supply:", bobPercent.toFixed(2) + "%");
        console.log("Rate (tokens/USDC):", Number(bobBalance) / Number(buyAmount));

        // Calculate and display the advantage
        const advantage = (Number(aliceBalance) / Number(launchAmount)) / (Number(bobBalance) / Number(buyAmount));
        console.log("\nPrice Advantage:");
        console.log("Launch price advantage multiplier:", advantage.toFixed(2) + "x");
        
        // Verify launcher gets significantly more tokens
        expect(Number(aliceBalance)).to.be.gt(Number(bobBalance));
        expect(advantage).to.be.gt(1);
        
        console.log("\n=== Launch Price Advantage Test Complete ===");
    });
});
```

The console outputs are shown below.

```
=== Testing Launch Price Advantage ===

Launch Statistics:
Total Supply: 1000000.0
Alice's USDC spent: 20000.0
Alice's tokens received: 861239.592969472710453284
Alice's % of total supply: 86.12%
Rate (tokens/USDC): 43.06197964847364

First Buy Statistics:
Bob's USDC spent: 20000.0
Bob's tokens received: 65980.203245956692749045
Bob's % of total supply: 6.60%
Rate (tokens/USDC): 3.2990101622978347

Price Advantage:
Launch price advantage multiplier: 13.05x

=== Launch Price Advantage Test Complete ===
```

## Impact
There will be an unfair token distribution as the token deployer can purchase an unlimited amount of tokens at the launch price. This creates issues such as:
- Centralized token distribution
- Significant price advantage for a large supply of tokens at initial purchase
- Risk of supply manipulation by the deployer
- Potential for large sell pressure from concentrated holdings

## Recommendation
Implement a limit on the launch function that limits the amount of the deployed tokens the deployer can purchase.
