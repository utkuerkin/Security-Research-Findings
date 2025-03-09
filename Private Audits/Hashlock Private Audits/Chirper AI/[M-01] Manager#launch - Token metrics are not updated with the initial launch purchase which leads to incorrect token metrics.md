## Description
The `launch` function allows users to launch a new token and buy this token with the asset tokens they supply. This function sets the token metrics and then makes a call to the `router.buy` function to purchase tokens for the deployer. However, since the call is directly being made to `router.buy` instead of `manager.buy`, the initial purchase bypasses updating metrics, which leads to incorrect token metrics being shown. This can have serious implications and confusion for users and off-chain systems.

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

As observed in this function, it sets the initial token metrics and then makes a call to `router.buy`. 

### Comparing the `router.buy` and `manager.buy` functions below:

#### Manager.sol

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

#### Router.sol

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

As observed in the highlighted part, the `manager.buy` function that is callable by users updates token metrics, while the `router.buy` function that is called during the initial token launch does not update token metrics. This leads to token metrics being incorrect after the initial purchase during the launch.

## Impact
The token metrics will show incorrect data after the initial token purchase during the launch. This will have serious consequences for users and off-chain systems and will cause confusion.

## Recommendation
Move the `_updateMetrics` call into the `router.buy` function.

Alternatively, after the initial purchase of the token, manually call the `_updateMetrics` function to update token metrics.
