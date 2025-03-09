## Description
BAD token imposes a maximum wallet size limit on its users except for select EOAs and important contracts. However, the base liquidity pool is not exempt from this limit in the constructor which can disrupt swaps. This address can be set as exempt from this limit but this can be overlooked which will impact the token's trading.

## Vulnerability Details
The `getExceedsSizeLimit` function is used in the `_update` function to see if the result of the transaction will cause the receiving address to exceed the maximum wallet size limit.
```solidity
function getExceedsSizeLimits(address from, address to, uint256 amount) internal view returns (bool status) {
    if (isLimitExempt[from] || isLimitExempt[to]) {
        return false;
    }
    if (amount > maximumTransactionSize || (balanceOf(to) + amount) > maximumWalletSize) {
        return true;
    }
}
```
Take a look at the `constructor`.

```solidity
constructor(
    address _initialOwner,
    string memory _name,
    string memory _symbol,
    uint256 _maxSupply,
    address payable _revenueRecipient,
    address _lpTokenRecipient,
    address _uniswapV2Router02
) ERC20(_name, _symbol) ERC20Permit(_name) Ownable(_initialOwner) {
    /// Set transaction limits
    uint256 maxSupply = _maxSupply * 10 ** decimals();
    uint256 onePercentOfTotalSupply = intoUint256(ud(maxSupply).mul(ud(0.01e18)));
    maximumTransactionSize = onePercentOfTotalSupply;
    maximumWalletSize = onePercentOfTotalSupply;

    isLimitExempt[address(this)] = true;
    isLimitExempt[address(0)] = true;
    isLimitExempt[_initialOwner] = true;
    isLimitExempt[_uniswapV2Router02] = true;
    // rest of the function
}
```

It is observed that while some addresses are kept exempt from the limit, the base liquidity pool is not. To make sure no issues arise, the base liquidity pool should be kept exempt from the limit.

## Impact
The base liquidity pool will not be able to exceed 1% of the total supply limit put on any address. This will seriously affect the trading of the token.

## Recommendation
Set the `basePair` as `isLimitExempt`.
