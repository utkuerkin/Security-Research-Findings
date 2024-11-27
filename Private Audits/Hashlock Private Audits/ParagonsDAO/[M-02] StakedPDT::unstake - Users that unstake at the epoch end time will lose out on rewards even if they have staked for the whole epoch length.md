## Description
The `unstake` function will revert if the current block time is greater than the epoch end
time. However, this function does not revert if the block time is the same as the epoch
end time. Users that unstake right as the epoch ends will miss out on rewards.
Additionally, these rewards won’t be distributed to other stakers and will be left in the
contract.

## Vulnerability Details
The unstake function shown below reverts when the `block.timestamp` is greater than
the `epoch.endTime`.
```solidity
function unstake(address to, uint256 amount) external nonReentrant {
    Epoch memory _epoch = epoch[currentEpochId];
    if (block.timestamp > _epoch.endTime) revert OutOfEpoch();
}
```
As seen in this function, if `block.timestamp` is equal to the `epoch.endTime`, the function
will not revert. Users that staked the whole duration of the epoch can unstake just as the epoch ends, this will result in them losing out on their rewards even though they
have staked the full duration of an epoch. Rewards that these users should have earned
will be stuck in the contract until they are withdrawn by a privileged role.

## Proof of Concept
Implement and run the following test in the `StakedPDT_claim.t.sol` file.
```solidity
function test_unstakeAtEpochEnd() public {
    uint256 POOL_SIZE = 1e18;
    uint256 initialBalance = 1e18;
    bPDTOFT.mint(staker1, initialBalance);
    bPDTOFT.mint(staker2, initialBalance);
    /// EPOCH 0
    _creditPRIMERewardPool(POOL_SIZE);
    _moveToNextEpoch(0);
    /// EPOCH 1
    // users stake
    vm.startPrank(staker1);
    bPDTOFT.approve(bStakedPDTAddress, initialBalance);
    bStakedPDT.stake(staker1, initialBalance);
    vm.stopPrank();
    vm.startPrank(staker2);
    bPDTOFT.approve(bStakedPDTAddress, initialBalance);
    bStakedPDT.stake(staker2, initialBalance);
    vm.stopPrank();
    // advance time to the end of epoch 1
    (, uint256 epochEndTime,) = bStakedPDT.epoch(1);
    vm.warp(epochEndTime);
    //staker1 unstakes at epoch end
    vm.startPrank(staker1);
    bStakedPDT.unstake(staker1, initialBalance);
    vm.stopPrank();
    // move to epoch 2
    _creditPRIMERewardPool(POOL_SIZE);
    _moveToNextEpoch(1);
    // stakers claim rewards
    vm.startPrank(staker1);
    bStakedPDT.claim(staker1);
    vm.stopPrank();
    vm.startPrank(staker2);
    bStakedPDT.claim(staker2);
    vm.stopPrank();
    // logs
    console.log("Staker1 Reward Tokens:", bPRIME.balanceOf(staker1));
    console.log("Staker2 Reward Tokens:", bPRIME.balanceOf(staker2));
    console.log("Reward tokens in the contract:", bPRIME.balanceOf(bStakedPDTAddress));
}
```
Observing the logs we can see that the staker1 address has not received any rewards
even though they have staked for the full epoch duration. The rewards they have
earned are not distributed to other stakers and they are left in the staking contract.
```javascript
Staker1 Reward Tokens: 0
Staker2 Reward Tokens: 500000000000000000
Reward tokens in the contract: 1500000000000000000
```

## Impact
Users that have staked for the whole epoch duration can lose out on rewards. These
rewards are not distributed to other users and are left in the staking contract

## Recommendation
Change the if check as shown below in the `unstake` function.
```solidity
function unstake(address to, uint256 amount) external nonReentrant {
    Epoch memory _epoch = epoch[currentEpochId];
    if (block.timestamp >= _epoch.endTime) revert OutOfEpoch();
}
```
This change will make sure that the unstaking is not possible when epoch ends.

Update the contract to make sure that the rewards unstaking users should’ve gotten are
distributed among other stakers.
