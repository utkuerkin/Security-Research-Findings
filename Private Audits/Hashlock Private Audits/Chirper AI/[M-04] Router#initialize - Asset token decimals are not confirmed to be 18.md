## Description
The calculations involving the agent token and asset token always assume these tokens have the same decimals. If asset tokens that have decimals less than 18 are used, it results in the undervaluing of this asset token during price calculations. Likewise, if asset tokens that have decimals more than 18 are used, it results in overvaluing this asset token during price calculations. Since the asset token is set during the initialization of the `Router.sol` contract and there are no checks done to make sure this asset token has 18 decimals, this can seriously affect price calculations.

## Vulnerability Details
The `initialize` function sets the asset token that will be used for tokens that are launched.

```solidity
function initialize(
    address factory_,
    address assetToken_
) external initializer {
    __ReentrancyGuard_init();
    __AccessControl_init();
    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);

    require(factory_ != address(0), "Invalid factory");
    require(assetToken_ != address(0), "Invalid asset token");

    factory = Factory(factory_);
    assetToken = assetToken_;
}
```

This asset token is used in the price calculations in the `Manager#launch` and `Router#_getAmountsOut` functions.

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

    uint256 k = ((K * 10000) / assetRate);
    uint256 liquidity = (((k * 10000 ether) / supply) * 1 ether) / 10000;

    router.addInitialLiquidity(address(actualToken), supply, liquidity);

    // rest of the function
}
```

```solidity
function _getAmountsOut(
    address token_,
    address assetToken_,
    uint256 amountIn_
) internal view returns (uint256) {
    require(token_ != address(0), "Invalid token");

    address pairAddress = factory.getPair(token_, assetToken);
    IPair pair = IPair(pairAddress);

    (uint256 reserveA, uint256 reserveB) = pair.getReserves();
    uint256 k = pair.kLast();

    if (assetToken_ == assetToken) {
        uint256 newReserveB = reserveB + amountIn_;
        uint256 newReserveA = k / newReserveB;
        return reserveA - newReserveA;
    } else {
        uint256 newReserveA = reserveA + amountIn_;
        uint256 newReserveB = k / newReserveA;
        return reserveB - newReserveB;
    }
}
```

As observed in these functions, no decimal normalization is done for the tokens and it is simply assumed that the asset token decimals are the same as the agent token decimals (18). However, if the asset token decimals are different, for example, 6 decimals for USDC, it would result in serious undervaluation of this asset token. In short, the protocol does not account for different decimals in calculations, and all asset tokens must have the same decimals as the agent tokens being launched in order for the calculations to work.

## Impact
When asset tokens with decimals different than 18 are used, it will result in serious over/undervaluation of these asset tokens during price calculations.

## Recommendation
Implement a check in the `initialize` function to make sure that the asset token decimals are 18. An example is shown below.

```solidity
function initialize(
    address factory_,
    address assetToken_
) external initializer {
    __ReentrancyGuard_init();
    __AccessControl_init();
    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);

    require(factory_ != address(0), "Invalid factory");
    require(assetToken_ != address(0), "Invalid asset token");
    require(IERC20(assetToken_).decimals() == 18, "Must have 18 decimals");
    factory = Factory(factory_);
    assetToken = assetToken_;
}
```
