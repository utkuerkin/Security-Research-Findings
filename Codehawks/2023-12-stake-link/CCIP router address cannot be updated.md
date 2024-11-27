### Sponsor: stake.link

### Dates: Dec 22nd, 2023 - Jan 12th, 2024

[See more contest details here](https://www.codehawks.com/contests/clqf7mgla0001yeyfah59c674)


	


# Low Risk Findings

## <a id='L-01'></a>L-01. CCIP router address cannot be updated            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol

https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol

https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/WrappedTokenBridge.sol

https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/base/SDLPoolCCIPController.sol

## Summary

CCIP Router addresses cannot be updated in `SDLPoolCCIPController.sol, SDLPoolCCIPControllerPrimary.sol, SDLPoolCCIPControllerSecondary.sol, WrappedTokenBridge.sol` . 

## Vulnerability Details

On contracts that inherit from `CCIPReceiver`, router addresses need to be updateable. Chainlink may update the router addresses as they did before. This issue introduces a single point of failure that is outside of the protocol's control.

[An example contract](https://github.com/smartcontractkit/ccip-tic-tac-toe/blob/main/contracts/TTTDemo.sol#L81-L83) that uses CCIP. [Taken from Chainlink docs](https://docs.chain.link/ccip/examples#ccip-tic-tac-toe).

[Chainlink documents noticing users about router address updating on testnet.](https://docs.chain.link/ccip/release-notes#v120-release-on-testnet---2023-12-08)

> CCIP v1.0.0 has been deprecated on testnet. You must use the new router addresses mentioned in the [CCIP v1.2.0 configuration page](https://docs.chain.link/ccip/supported-networks/v1_2_0/testnet) **before January 31st, 2024**
> 

On Testnets, router contracts in v1.0.0 and v1.2.0 are different. It means that router contract addresses can change from version to version. So CCIPReceivers should accommodate this. Mainnet is on v1.0.0 which means its router addresses can change with an update.

## Impact

Impact: High
Likelihood: Low

Router address deprecation will cause the protocol to stop working.

## Tools Used

Manual review.

## Recommendations

Implement a function to update the `_router` address. Example shown below:

```jsx
function updateRouter(address routerAddr) external onlyOwner {
        _router = routerAddr;
    }
```
