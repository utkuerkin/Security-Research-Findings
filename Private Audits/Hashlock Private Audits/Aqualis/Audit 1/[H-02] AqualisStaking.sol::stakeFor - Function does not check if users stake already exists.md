## Description
The `stakeFor` function allows the owner to stake for other users. However, the lack of
checks causes this function to be vulnerable.

## Vulnerability Details
The `stakeFor` function lacks any checks to see if the staked-for user has an existing
stake. When this function is called it will override the user's stake if they already are
staked. All other staking functions in the contract check if the user has an existing
stake.
```solidity
function stakeFor(address _staker, uint256 _amount, uint8 _weeksNum) external onlyOwner {
    aqualisToken.transferFrom(_msgSender(), address(this), _amount);
    _createStake(_staker, _amount, _weeksNum);
}
```

## Impact
Malicious owner can override any user's existing stake to essentially delete it, this
creates a centralization risk.

Additionally, the owner can mistakenly override a user’s existing stake.

## Recommendation
Implement the following check into the `stakeFor` function 
```solidity
function stakeFor(address _staker, uint256 _amount, uint8 _weeksNum) external onlyOwner {
    require(stakers[_staker].amount == 0, "User already has active staking");
    aqualisToken.transferFrom(_msgSender(), address(this), _amount);
    _createStake(_staker, _amount, _weeksNum);
}
```
