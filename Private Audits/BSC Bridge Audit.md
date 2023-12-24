# Bridge Report

# Introduction

A time-boxed security review of the **redacted** protocol was done with a focus on the security aspects of the application’s smart contracts implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

## Privileged Roles & Actors

- **Proxy Admin:** Upgrades the proxy contract to a new implementation contract.
- **Bridge Script Admin:** We assume this role is assigned to an account managed by the off-chain Bridge server to automatically run the on-chain bridge tasks.
- **Bridge Manual Admin:** This role has the same privileges as the Bridge Script Admin. **:** We assume this role is used for manually altering the bridge state.
- **Bridge Manage Admin:** Manages the bridge parameters. Can add/remove tokens, chains and other relevant parameters.
- **LP Provider:** Provides liquidity to the Bridge. Earns fees in return.

# Severity classification

| Severity | Impact: High | Impact: Medium | Impact: Low |
| --- | --- | --- | --- |
| Likelihood: High | Critical | High | Medium |
| Likelihood: Medium | High | Medium | Low |
| Likelihood: Low | Medium | Low | Info |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

# Security Assessment Summary

### Scope

The following smart contracts were in scope of the audit:

- **redacted**
- **redacted**

---

# Findings Summary

| ID | Title | Severity | Status |
| --- | --- | --- | --- |
| [H-01] | Incorrect feeForBridgeGas for different blockchains | High | Not Fixed |
| [H-02] | withdrawLiquidity function doesn’t update the balances.  | High | Not Fixed |
| [H-03] | bridgeOut function transfers funds without fees in some cases | High | Not Fixed |
| [M-01] | Token transfer' does not handle case if the tokens support fee-on-transfer | Medium | Not Fixed |
| [M-02]  | Multiple centralization attack vectors are present in the protocol | Medium | Not Fixed |
| [L-01] | Missing _disableInitializers(); in the implementation contract’s constructor | Low | Not Fixed |
| [I-01] |  Different tokens have different decimals | Info | Not Fixed |
| [I-02]  | Lack of modifier usage | Info | Not Fixed |
| [I-03] | Typo in lines 140 and 264. | Info | Not Fixed |
| [I-04] | updateFeeand updateAdminWallets functions should be separated. | Info | Not Fixed |
| [I-05]  | Use constants instead of raw values | Info | Not Fixed |
| [I-06] | Lack of NatSpec development comments | Info | Not Fixed |
| [I-07] | Lack of event emits | Info | Not Fixed |
| [I-08] | Tokens aren’t removed from mapping when a chain is removed | Info | Not Fixed |
| [I-09] | Use if and revert with custom errors instead of require | Info | Not Fixed |
| [I-10] | updateFee function does not have upper limit | Info | Not Fixed |

# Detailed Findings

# [H-01] Incorrect `feeForBridgeGas` for different blockchains

## Severity

**Impact:** Medium, users will pay extra gas fees if they bridge to a low-fee chain.  

**Likelihood:** High, every time the bridge is used this will occur.  

## Description

`feeForBridgeGas` should be different for each chain. This isn’t a static value and each chain has different avg. gas price. For example, on Polygon USDT transfer costs around 0.003 USD  while on Ethereum it costs 1.877 USD. 

`feeForBridgeGas` is set within the `updateFee` function (Line 86-93) and It is used as the same value for every chain. This will lead to higher than expected `feeForBridgeGas` for blockchains that have low gas costs since `feeForBridgeGas` should be set according to the blockchain with the highest gas cost (at the moment this will probably be the Ethereum chain.) in order to keep the contract from losing money to pay the gas fees. `feeForBridgeGas` set this way will make it so that a user will have to pay way too much gas fee than they would need to pay if they are using a cheaper blockchain.

## Recommendations

Set different gas fees for different bridges. This can be done by using an oracle to take the average gas price at that time and setting is as that blockchains gas fee.

# [H-02] `withdrawLiquidity` function doesn’t update the `totalBalance` or `totalLPBalance`.

## Severity

**Impact:** High, using this function can leave the balance in the wrong state. 

**Likelihood:** Medium, only admins can call this function so they might take extra precautions to prevent protocol from going to a bad state.

## Description

`withdrawLiquidity` function doesn’t update the `totalBalance` or `totalLPBalance`.  The function should update these two mappings in order to avoid storing incorrect state of the balance.

## Recommendations

Add the correct update logic to the code:

```jsx
function withdrawLiquidity(address owner_, address _token, uint256 amount, uint256 rootChain) external nonReentrant {
        require(_msgSender() == bridgeManualAdmin || _msgSender() == bridgeScriptAdmin, "not allowed");
        require(isTokenListed[_token]>0, "not existed");
        if(_token == address(0x1)){
            if(address(this).balance>=amount){
								totalBalance[_token] -= amount;
								totalLPBalance[_token] -= amount;
                (bool sent,) = payable(owner_).call{value:amount}("");
                require(sent, "Failed to send Ether");        
                emit WithdrawLiquidity(owner_, tokensInOtherChains[_token][rootChain], amount, rootChain);
            }else{
                uint256 _amount = address(this).balance;
                if(_amount > 0){
										totalBalance[_token] -= _amount;
										totalLPBalance[_token] -= _amount;
                    (bool sent,) = payable(owner_).call{value:_amount}("");
                    require(sent, "Failed to send Ether");  
                    emit WithdrawLiquidity(owner_, tokensInOtherChains[_token][rootChain], _amount, rootChain);
                }        
                address[] memory _tokens;
                uint256[] memory _chains;
                uint256 count=0;
                for(uint256 i=0;i<chains.length;i++){
                    if(tokensInOtherChains[_token][chains[i]] != address(0)){
                        _tokens[count]=tokensInOtherChains[_token][chains[i]];
                        _chains[count]=chains[i];
                    }
                    
                }    
                emit LiquidityRequiredInOtherChain(owner_, _tokens, _chains, amount-_amount, rootChain);
            }
```

# [H-03]  `bridgeOut` function transfers funds without fees in some cases

## Severity

**Impact:** High, users will bridge out tokens without paying LP and admin fees.  

**Likelihood:** Medium, this only works if the destination chain doesn’t have enough balance to transfer all the user’s funds.

## Description

In `bridgeOut` function, if the released token amount is not sufficient for full release of the bridged tokens, the bridge will send all the available balance to the receiver. In that case, bridge contract calculates the funds as follows:

```jsx
}else{
                uint256 _amount = address(this).balance;
                amountForAdmin = _amount * _tokenFeeForAdmin / (1000000 - _tokenFeeForAdmin - _tokenFeeForLPProvider);
                amountForLPProvider = _amount * _tokenFeeForLPProvider / (1000000 - _tokenFeeForAdmin - _tokenFeeForLPProvider);
                feeCollectedForAdmin[_token] += amountForAdmin;
                totalBalance[_token] += amountForLPProvider;
                if(_amount > 0){
                    (bool sent,) = payable(to).call{value:_amount}("");
                    require(sent, "Failed to send Ether");
                    emit BridgeOut(sender, to, _token, rootChain, _amount);
                }
```

Here the `amountForAdmin` and `amountForLPProvider` fees are not subtracted from the  transferred funds `_amount`. This leads to transferring the bridge funds without any fees. Also `feeCollectedForAdmin` and `amountForLPProvider` mappings will have a wrong state as all the available funds are sent to the user.

Example attack scenario:

1. User has 10,000.01 USDC tokens in BSC and wants to bridge that to Ethereum.
2. User sees in the Bridge contract in Ethereum there are only 10,000.00 USDC tokens.
3.  User initiates the bridgeIn process in BSC.
4. After a while, bridgeOut function will be invoked by the bridge admins.
5. Since the user transferred amount is higher than the available balance in bridge ( 10,000.01 > 10,000.00) , user will be sent all the available funds.
6. User will obtain 10,000.00 USDC with only losing 1 cent. 
7. Whereas if the protocol had contained sufficient funds then the user would have paid much more with the LP and admin fees. 

## Recommendations

Make sure to subtract the fees from the `_amount` before the funds are transferred to the user:

```jsx
_amount = _amount - amountForAdmin - amountForLPProvider;
```

# [M-01] **Token transfer' does not handle case if the tokens support fee-on-transfer**

## Severity

**Impact:** High, bridge LP providers will lose funds.  

**Likelihood:** Low,  admins need to whitelist a fee-on-transfer token which are not that common.  

## Description

Some tokens take a transfer fee (e.g. STA, PAXG), some do not currently charge a fee but may do so in the future (e.g. USDT, USDC). 

While transfer of an ERC20 token to the bridge contract, In the current implementation, it is assumed that the received amount is the same as the transfer amount. However, due to how fee-on-transfer tokens work, much less will be received than what was transferred.

For example, in the `bridgeIn` function following line of code assumes the transferred amount will be equal to the user supplied  `amount` :

```jsx
require(tokensInOtherChains[_token][_chain]!=address(0), "no token registered");
        if(_token != address(0x1))
            IERC20Upgradeable(_token).safeTransferFrom(_msgSender(), address(this), amount);
        bridgeNonce[_token] += 1;
```

This will lead to loss of funds to the bridge LP providers.

## Recommendations

I am not aware of any legitimate uses of fee-on-transfer tokens, so I would suggest not supporting them. So when first adding tokens to the system verify that the balance changed by the expected amount following a transfer so that fee-on-transfer tokens cannot enter.

In order to obtain the actual amount received by the contract, track the balance of tokens before and after the transfer of tokens. For example, in the contract test, we recommend implementing the following steps:

```jsx
function _transfer(uint256 amount) public returns(uint256)
{
	uint256 balanceBefore = IERC20(token).balanceOf(address(this));
	IERC20Token(token).SafetransferFrom(msg.sender, address(this),amount);

	uint256 balanceAfter = IERC20(token).balanceOf(address(this));
	require(balanceAfter >= balanceBefore);
	return balanceAfter - balanceBefore;
}
```

# [M-02] **Multiple centralization attack vectors are present in the protocol**

## Severity

**Impact:** High, as it can result in a rug from the protocol owner

**Likelihood:** Low, as it requires a compromised or a malicious owner

## Description

The protocol owner has privileges to control the funds in the protocol or the flow of them. Admin roles (`bridgeScriptAdmin, bridgeManualAdmin, bridgeManageAdmin`) have too much centralized power over the protocol. A malicious admin can use this authority to steal money from the contract and forcefully liquidate LP’s. `bridgeManualAdmin` and `bridgeScriptAdmin` roles have the power to call `withdrawLiquidity` function (Line 206-257) to send tokens from the contract to any address they want without any checks. Same roles can also call `forceRemoveLiquidity` (Line 259-270) function to forcefully liquidate any LP they want without any checks or reasoning for the forceful liquidation of an LP. 

## Recommendations

Consider removing some owner privileges or put them behind a Timelock contract or governance.

# [L-01] Missing `_disableInitializers();` in the implementation contract’s constructor

## Severity

**Impact:** Low, anyone can takeover the implementation contract but they cannot affect the proxy contract.

**Likelihood:** Low, attackers got nothing to gain from this.  

## Description

Missing the `_disableInitializers()` function in the implementation contract’s constructor. An uninitialized contract can be taken over by an attacker. This applies to both a proxy and its implementation contract, which may impact the proxy. In this case, the proxy contract is will. be initialized however the implementation contract is not. To prevent the implementation contract from being used, you should invoke the `_disableInitializers()` function in the constructor to automatically lock it when it is deployed.

## Recommendations

Implement `_disableInitializers()` function in contract’s constructor.

```jsx
contract Bridge is Initializable, OwnableUpgradeable, ReentrancyGuardUpgradeable {
    constructor(){
        _disableInitializers();
    }
```

# [I-01] Different tokens have different decimals

## Severity

**Impact:** Informational

## Description

Some ERC20 tokens have different `decimals` on different chains. Even some popular ones like USDT and USDC have 6 decimals on Ethereum, and 18 decimals on BSC for example:

- [USDT on Ethereum](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#readContract#F6) - 6 decimals
- [USDC on Ethereum](https://etherscan.io/token/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48#readProxyContract#F11) - 6 decimals
- [USDT on BSC](https://bscscan.com/address/0x55d398326f99059ff775485246999027b3197955#readContract#F6) - 18 decimals
- [USDC on BSC](https://bscscan.com/address/0x8ac76a51cc950d9822d68b83fe1ad97b32cd580d#readProxyContract#F3) - 18 decimals

## Recommendations

Different ERC20 tokens having different decimals should be taken into consideration.

# [I-02] Lack of modifier usage

## Severity

**Impact:** Informational

## Description

In the contract, in order to check a condition in certain functions, contract uses `require` . For example:

```
require(_msgSender() == bridgeManageAdmin, "not allowed");
require(isChainListed[_chain]==0, "already existed");
```

But It would be better practice to use modifiers for the functions to increase readability of the code and make it easier to change these modifiers if needed. In order to change these modifiers we would need to change only few lines of code instead of changing it in every single function.

## Recommendations

Instead of using require in every function we need to check a condition for(For example: Line 68-69).

```
function removeChain(uint256 _chain) external {
        require(_msgSender() == bridgeManageAdmin, "not allowed");
        require(isChainListed[_chain]>0, "not existed");
        chains[isChainListed[_chain]-1] = chains[chains.length-1];
        isChainListed[chains[chains.length-1]] = isChainListed[_chain];
        delete isChainListed[_chain];
        chains.pop();      
    }
```

we can instead make custom modifiers such as:

```jsx
error Bridge_MissingAuthorization();
error Bridge_ChainDoesNotExist();

modifier onlyBridgeManageAdmin() {
        if{
					_msgSender() == bridgeManageAdmin;
				revert Bridge_MissingAuthorization();
				}
        _;
}

    modifier chainExists(uint256 _chain) {
        if{
					isChainListed[_chain] > 0;
					revert Bridge_ChainDoesNotExist;
				}
        _;
}

    function removeChain(uint256 _chain) external onlyBridgeManageAdmin chainExists(_chain){
	      chains[isChainListed[_chain]-1] = chains[chains.length-1];
        isChainListed[chains[chains.length-1]] = isChainListed[_chain];
        delete isChainListed[_chain];
        chains.pop();
}
```

And apply these modifiers into our function as shown in the example.

# [I-03] Typo in lines 140 and 264

## Severity

**Impact:** Informational

## Description

Typo in lines 140 and 264(`enogh` instead of `enough`)

```jsx
require(LPBalanceOf[_token][_msgSender()] >= LPBalance, "You don't have enogh Liquidity");
```

## Recommendations

Change the lines to:

```jsx
require(LPBalanceOf[_token][_msgSender()] >= LPBalance, "You don't have enough Liquidity");
```

# [I-04] `updateFee`and `updateAdminWallets` functions should be separated

## Severity

**Impact:** Informational/Gas

## Description

`updateFee`and `updateAdminWallets` functions should be separated to have a single function for each parameter. 

```jsx
function updateFee(uint256 _feeForBridgeGas, uint256 _feeForAdmin, uint24 _tokenFeeForAdmin, uint24 _tokenFeeForLPProvider) external {
        require(_msgSender() == bridgeManageAdmin, "not allowed");
        feeForBridgeGas = _feeForBridgeGas;
        feeForAdmin = _feeForAdmin;
        tokenFeeForAdmin = _tokenFeeForAdmin;
        tokenFeeForLPProvider = _tokenFeeForLPProvider;
        emit FeeUpdated(feeForBridgeGas, feeForAdmin, tokenFeeForAdmin, tokenFeeForLPProvider);
    }
```

Even if we want to change only one value such as `feeForBridgeGas` only way for us to do this is calling the `updateFee` function and reentering all the other parameters `uint256 _feeForBridgeGas, uint256 _feeForAdmin, uint24 _tokenFeeForAdmin, uint24 _tokenFeeForLPProvider` alongside the only parameter we want to change. Same goes for the `updateAdminWallets` function. 

```
function updateAdminWallets(address _bridgeScriptAdmin, address _bridgeManualAdmin, address _bridgeManageAdmin, address _treasury) external onlyOwner{
        bridgeScriptAdmin = _bridgeScriptAdmin;
        bridgeManualAdmin = _bridgeManualAdmin;
        bridgeManageAdmin = _bridgeManageAdmin;
        treasury = _treasury;
    }
```

This will lead to us using more gas than necessary each time we want to update a value.

## Recommendations

Make separate functions to update all the parameters that are shown above. `updateAdminWallets` can be separated as such:

```
function updateBridgeScriptAdmin(address _bridgeScriptAdmin) external onlyOwner {
        bridgeScriptAdmin = _bridgeScriptAdmin;
}

    function updateBridgeManualAdmin(address _bridgeManualAdmin) external onlyOwner {
        bridgeManualAdmin = _bridgeManualAdmin;
}

    function updateBridgeManageAdmin(address _bridgeManageAdmin) external onlyOwner {
        bridgeManageAdmin = _bridgeManageAdmin;
}

    function updateTreasury(address _treasury) external onlyOwner {
        treasury = _treasury;
}
```

`updateFee` can be separated as such:

```jsx
function updateFeeForBridgeGas(uint256 _feeForBridgeGas) external {
        require(_msgSender() == bridgeManageAdmin, "not allowed");
        feeForBridgeGas = _feeForBridgeGas;
        emit FeeForBridgeGasUpdated(_feeForBridgeGas);
}

    function updateFeeForAdmin(uint256 _feeForAdmin) external {
        require(_msgSender() == bridgeManageAdmin, "not allowed");
        feeForAdmin = _feeForAdmin;
        emit FeeForAdminUpdated(_feeForAdmin);
}

    function updateTokenFeeForAdmin(uint24 _tokenFeeForAdmin) external {
        require(_msgSender() == bridgeManageAdmin, "not allowed");
        tokenFeeForAdmin = _tokenFeeForAdmin;
        emit TokenFeeForAdminUpdated(_tokenFeeForAdmin);
}

    function updateTokenFeeForLPProvider(uint24 _tokenFeeForLPProvider) external {
        require(_msgSender() == bridgeManageAdmin, "not allowed");
        tokenFeeForLPProvider = _tokenFeeForLPProvider;
        emit TokenFeeForLPProviderUpdated(_tokenFeeForLPProvider);
}
```

This will cost more gas to deploy the contract but It will take less gas when we don’t need to change all the values.

# [I-05] Use constants instead of raw values

## Severity

**Impact:** Informational

## Description

In the `bridgeOut`function (Line 294-351). The value “1000000” is used 10 separate times. In order to increase code readability we recommend creating a `constant` and assign the value “1000000” to it.

## Recommendations

Create a constant like `uint256 private constant PRECISION = 1_000_000;` and replace the values in the code with it.

# [I-06] Lack of NatSpec development comments

## Severity

**Impact:** Informational

## Description

It is recommended that development teams follow the best practices for leveraging NatSpec when creating documentation to communicate the functionalities and constraints of your smart contracts more effectively.

It is also recommended to have a set layout of the contract to increase readability.

## Recommendations

Use of NatSpec tags such as `@dev` , `@notice` ,`@param` and comments should be used to give contextual descriptions, explain the parameters taken in a function and explain the expected use of a function. For example we can add these comments to the `addLiquidity` function:

```jsx
//@notice This is the function where an LP can add liquidity to the contract
//@param _token The address of the token to add as liquidity
//@param amount The amount of _token to be added as liquidity
function addLiquidity(address _token, uint256 amount) external payable {
        require(isTokenListed[_token]>0, "not existed");
        if(_token == address(0x1)){
            require(msg.value >= amount, "Insufficient ETH");      
        }else{
            IERC20Upgradeable(_token).safeTransferFrom(_msgSender(), address(this), amount);
        }        
        uint256 LPBalance = totalBalance[_token] > 0 ? amount * totalLPBalance[_token] / totalBalance[_token] : amount;
        totalBalance[_token] += amount;
        totalLPBalance[_token] += LPBalance;
        LPBalanceOf[_token][_msgSender()] += LPBalance;
        emit AddLiquidity(_msgSender(), _token, amount, totalBalance[_token], totalLPBalance[_token], LPBalanceOf[_token][_msgSender()]);
    }
```

A layout of the contract as shown below will be helpful for the developers and the auditors. Contract should be arranged in the order of the layout.

```jsx
//SPDX-License-Identifier: MIT

// Layout of Contract:
// version
// imports
// interfaces, libraries, contracts
// errors
// Type declarations
// State variables
// Events
// Modifiers
// Functions

// Layout of Functions:
// constructor
// receive function (if exists)
// fallback function (if exists)
// external
// public
// internal
// private
// view & pure functions

pragma solidity 0.8.19;
import "@openzeppelin/contracts-upgradeable/token/ERC20/utils/SafeERC20Upgradeable.sol";
import "@openzeppelin/contracts-upgradeable/token/ERC20/extensions/IERC20MetadataUpgradeable.sol";

import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/security/ReentrancyGuardUpgradeable.sol";

contract Bridge is Initializable, OwnableUpgradeable, ReentrancyGuardUpgradeable {
```

# [I-07] Lack of event emits

## Severity

**Impact:** Informational

## Description

- Lack of event emits in these functions:
    - `addToken`
    - `removeToken`
    - `includeFeeForToken`
    - `excludeFeeForToken`
    - `addChain`
    - `removeChain`
    - `updateAdminWallets`
    - `setTokenForOtherChain`

## Recommendations

Add event emits to the functions like in example:

```jsx
event TokenAdded(address sender, address token, uint256 amount);

function addToken(address _token, string memory _logo) external {
        require(_msgSender() == bridgeManageAdmin, "not allowed");
        require(isTokenListed[_token]==0, "already existed");
        tokens.push(_token);
        logos[_token]=_logo;
        isTokenListed[_token] = tokens.length;
				emit TokenAdded(_msgSender(), _token, _logo);
    }
```

# [I-08] Tokens aren’t removed from mapping when a chain is removed

## Severity

**Impact:** Informational

## Description

Ensure that when the `removeChain` function is called and the chain is removed all tokens in that chain are removed as well. When the chain is removed, a token’s  `tokensInOtherChains`  value for that chain will not be deleted.

## Recommendations

Make sure to update the `removeChain` function to update the `tokensInOtherChains` .

# [I-09] Use `if` and `revert` with custom errors instead of `require`

## Severity

**Impact:** Informational/Gas

## Description

Using `custom errors` is a good way to improve readability and be more gas efficient. The contract should have `custom errors` and use `if` and`revert` instead of `require` .

## Recommendations

Use custom errors such as:

```
error Bridge_InsufficentFee();
error Bridge_TokenDoesntExist();
error Bridge_NoTokenRegistered();
```

With these custom errors we can change the `bridgeIn` function as such:

```jsx
function bridgeIn(address _token, uint256 _chain, address to, uint256 amount) external payable {
        if(_token == address(0x1)){
            if{
								msg.value >= feeForBridgeGas + feeForAdmin + amount;
					      revert Bridge_InsufficentFee();
						}
            (bool sent, ) = payable(bridgeScriptAdmin).call{value: feeForBridgeGas}("");
            require(sent, "Failed to send Ether");
            (sent, ) = payable(treasury).call{value: feeForAdmin}("");
            require(sent, "Failed to send Ether");
        }else{
            require(msg.value >= feeForBridgeGas + feeForAdmin, "Insufficient fee");        
            (bool sent, ) = payable(bridgeScriptAdmin).call{value: msg.value}("");
            require(sent, "Failed to send Ether");
            (sent, ) = payable(treasury).call{value: feeForAdmin}("");
            require(sent, "Failed to send Ether");
        }        
        if{
            isTokenListed[_token]>0;
            revert Bridge_TokenDoesntExist();
        }
        if{
						tokensInOtherChains[_token][_chain]!=address(0);
						revert Bridge_NoTokenRegistered();
        if(_token != address(0x1))
            IERC20Upgradeable(_token).safeTransferFrom(_msgSender(), address(this), amount);
        bridgeNonce[_token] += 1;
        emit BridgeIn(_msgSender(), tokensInOtherChains[_token][_chain], _chain, to, amount, bridgeNonce[_token]);
    }
```

# [I-10] `updateFee` function does not have upper limit

## Severity

**Impact:** Informational

## Description

`updateFee` function should have a limit on how high the fee can be in order to prevent admins from setting high fees to get all the users’ bridged funds.

## Recommendations

An `if` check can be added to make sure the entered parameters will have an upper limit.

```jsx
error Bridge_FeeUpperLimitExceeded();

function updateFee(uint256 _feeForBridgeGas, uint256 _feeForAdmin, uint24 _tokenFeeForAdmin, uint24 _tokenFeeForLPProvider) external {
        require(_msgSender() == bridgeManageAdmin, "not allowed");
				if (_feeForBridgeGas > feeLimit1 || _feeForAdmin > feeLimit2 || _tokenFeeForAdmin > feeLimit3 || _tokenFeeForLPProvider > feeLimit4) {
		    revert Bridge_FeeUpperLimitExceeded();
				}
        feeForBridgeGas = _feeForBridgeGas;
        feeForAdmin = _feeForAdmin;
        tokenFeeForAdmin = _tokenFeeForAdmin;
        tokenFeeForLPProvider = _tokenFeeForLPProvider;
        emit FeeUpdated(feeForBridgeGas, feeForAdmin, tokenFeeForAdmin, tokenFeeForLPProvider);
    }
```
