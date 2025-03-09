
## Description  
The `BedrockERC20` token inherits the `AdvancedERC20` contract that has voting mechanisms implemented in it. Bridging tokens can potentially lead to users utilizing their voting power twice on the L1 and L2 chains without acquiring new tokens.  

## Vulnerability Details  
The `__mint` function is called by the bridge when users bridge to L2:  

```solidity
function __mint(address to, uint256 value) private {
    // non-zero recipient address check
    require(to != address(0), "zero address");

    // non-zero value and arithmetic overflow check on the total supply
    // this check automatically secures arithmetic overflow on the individual balance
    unchecked {
        // put operation into unchecked block to display user-friendly overflow error message for Solidity 0.8+
        require(totalSupply + value > totalSupply, "zero value or arithmetic overflow");
    }

    // uint192 overflow check (required by voting delegation)
    require(totalSupply + value <= type(uint192).max, "total supply overflow (uint192)");

    // perform mint:
    // increase total amount of tokens value
    totalSupply += value;

    // increase `to` address balance
    tokenBalances[to] += value;

    // update total token supply history
    __updateHistory(totalSupplyHistory, add, value);

    // create voting power associated with the tokens minted
    __moveVotingPower(msg.sender, address(0), votingDelegates[to], value);

    // fire a minted event
    emit Minted(msg.sender, to, value);

    // emit an improved transfer event (arXiv:1907.00903)
    emit Transfer(msg.sender, address(0), to, value);

    // fire ERC20 compliant transfer event
    emit Transfer(address(0), to, value);
}
```

As observed in the highlighted part, users will receive voting power according to the amount that was minted.  

Think of the scenario where the user has 1000 tokens and for simplicity assume 1 token = 1 voting power.  
- User has 1000 tokens on L1, which is 1000 voting power.  
- User uses this voting power to vote on a certain DAO decision.  
- User then bridges their tokens to L2.  
- User gets minted 1000 tokens on L2, which is once again 1000 voting power.  
- User uses this voting power to vote on a certain DAO decision again.  

Depending on the voting mechanisms, this could allow users to double vote with their existing tokens.  

## Impact  
Users can double vote with their existing tokens.  

## Recommendation  
The voting mechanism should use a snapshot mechanic that will count a user’s both L1 and L2 tokens at that time when they try to vote.  
