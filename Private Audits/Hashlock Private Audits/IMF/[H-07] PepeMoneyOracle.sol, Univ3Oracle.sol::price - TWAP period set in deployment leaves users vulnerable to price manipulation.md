## Description
The `price` function in the oracle contracts calculates prices from Uniswap pools using
[TWAP](https://blog.uniswap.org/uniswap-v3-oracles#what-is-twap) (time weighted average price). TWAP is implemented to protect the Uniswap
oracle against price manipulation attacks. It is important to set the TWAP period
correctly.

## Vulnerability Details
The deployment script sets the `period` for TWAP to 60 seconds as shown below:
```solidity
contract DeployPepeMoneyOracle is Script {
    function run() external {
        vm.startBroadcast();
        new PepeMoneyOracle(60);
        vm.stopBroadcast();
    }
}
```
Referring to the [Uniswap v3 docs](https://blog.uniswap.org/uniswap-v3-oracles) most protocols set the TWAP period to 30 minutes to
protect their oracle against price manipulation.

## Impact
This vulnerability will leave the protocol and its users vulnerable to price manipulation
attacks.

## Recommendation
Set the TWAP period to 30 minutes in order to follow the most common practices.

