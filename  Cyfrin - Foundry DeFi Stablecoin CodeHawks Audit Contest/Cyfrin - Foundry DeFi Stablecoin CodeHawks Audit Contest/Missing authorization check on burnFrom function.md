### Severity

Medium Risk

### Relevant GitHub Links

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DecentralizedStableCoin.sol

# burnFrom Vuln.

## **Summary**

The issue lies in the **`burnFrom`** function of the DecentralizedStableCoin.sol, allowing users to bypass **`_burnDsc`** and burn tokens directly without proper checks**.** This manipulation results in an inaccurate **`s_DSCMinted`** value of the user. When we call **`burnFrom`** function the total supply of DSC will decrease but **`s_DSCMinted`** won’t reflect the changes where as it would correctly update with the intended burn functions.

## **Vulnerability Details**

DecentralizedStableCoin.sol inherits from 
ERC20Burnable.sol. ERC20Burnable implements two functions on top of 
ERC20 contract. These are burn and burnFrom. In DecentralizedStableCoin 
we have the onlyOwner modifier for burn function but no such checks for **`burnFrom`** function. Following test shows that the user can call burnFrom function.

```jsx
function testCanBurnFrom() public depositedCollateralAndMintedDsc {
        vm.startPrank(user);
        dsc.approve(user, amountToMint);
        dsc.burnFrom(user, amountToMint);
        vm.stopPrank();

        uint256 userBalance = dsc.balanceOf(user);
        assertEq(userBalance, 0);
    }
```

## **Impact**

Currently this vulnerability does not pose any significant
 security risk as it only affects the user that calls the burnFrom 
function. However any protocol that relies on DSC could get inaccurate 
balance and total supply data if they use the s_DSCMinted mapping. This 
vulnerability allows burner to bypass healthFactor checks that are 
implemented in intended burn function.

## **Tools Used**

Foundry, manual test.

## **Recommended Mitigation**

1. Override the burnFrom function in the DecentralizedStableCoin.sol and give it the onlyOwner modifier.

### Result
Validated