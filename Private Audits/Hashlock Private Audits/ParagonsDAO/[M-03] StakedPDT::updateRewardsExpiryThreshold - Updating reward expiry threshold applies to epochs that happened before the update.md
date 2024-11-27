## Description
The `updateRewardsExpiryThreshold` function updates the `rewardsExpiryThreshold`
variable which changes which epochs are eligible for rewards. Users that do not expect
this change will lose out on rewards when this variable is updated.

## Vulnerability Details
The `updateRewardsExpiryThreshold` function updates the `rewardsExpiryThreshold`
variable as shown below:
```solidity
function updateRewardsExpiryThreshold(uint256 newRewardsExpiryThreshold) external onlyRole(TOKEN_MANAGER) {
    if (newRewardsExpiryThreshold == 0) revert InvalidRewardsExpiryThreshold();
    rewardsExpiryThreshold = newRewardsExpiryThreshold;
    emit UpdateRewardDuration(newRewardsExpiryThreshold);
}
```
This function is backwards compatible, meaning that it will affect older epochs. Taking a
look at where the `rewardsExpiryThreshold` variable is used:
```solidity
function claim(address to) external nonReentrant {
    _setUserWeightAtEpoch(msg.sender);
    uint256 _currentEpochId = currentEpochId;
    uint256 _claimLeftOff = claimLeftOff[msg.sender];
    if (_claimLeftOff == _currentEpochId || _currentEpochId == 1) revert ClaimedUpToEpoch();
    uint256 _rewardsExpiryThreshold = rewardsExpiryThreshold;
    uint256 _startActiveEpochId =
        _currentEpochId > _rewardsExpiryThreshold ? _currentEpochId - _rewardsExpiryThreshold : 1;
    //rest of the function
}
```
This variable will make older epochs that are less than the `currentEpochId -
rewardsExpiryThreshold` ineligible for reward claims. For example: if the `currentEpochId`
is 25 and `rewardsExpiryThreshold`  is 15, epochs 1-10 will not be claimable.
Having no snapshots of older epochs and what the `rewardsExpiryThreshold` was when
users staked in these epochs will mean that this variable will apply to epochs before the
update.

## Proof of Concept
Implement and run the following test in the `StakedPDT_claim.t.sol` file.
```solidity
function test_updateRewardExpiryAfterStake() public {
    uint256 POOL_SIZE = 1e18;
    uint256 initialBalance = 1e18;
    bPDTOFT.mint(staker1, initialBalance * 2);
    bPDTOFT.mint(staker2, initialBalance * 2);
    bPDTOFT.mint(staker3, initialBalance * 2);
    //advance to epoch1
    _creditPRIMERewardPool(POOL_SIZE);
    _moveToNextEpoch(0);
    //staker1 stake at epoch1, reward expiry threshold is 24
    vm.startPrank(staker1);
    bPDTOFT.approve(bStakedPDTAddress, initialBalance * 2);
    bStakedPDT.stake(staker1, initialBalance);
    vm.stopPrank();
    //as the only staker in this epoch, staker1 should get all the epoch1 rewards.
    //advance to epoch2
    _creditPRIMERewardPool(POOL_SIZE);
    _moveToNextEpoch(1);
    //staker2 and staker3 stake at epoch2
    vm.startPrank(staker2);
    bPDTOFT.approve(bStakedPDTAddress, initialBalance * 2);
    bStakedPDT.stake(staker2, initialBalance);
    vm.stopPrank();
    vm.startPrank(staker3);
    bPDTOFT.approve(bStakedPDTAddress, initialBalance * 2);
    bStakedPDT.stake(staker3, initialBalance);
    vm.stopPrank();
    //advance to epoch3
    _creditPRIMERewardPool(POOL_SIZE);
    _moveToNextEpoch(2);
    //tokenManager updates reward expiry threshold to 1
    vm.startPrank(tokenManager);
    bStakedPDT.updateRewardsExpiryThreshold(1);
    vm.stopPrank();
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
    //showing that all claimed rewards are equal
    uint256 staker1Rewards = bPRIME.balanceOf(staker1);
    uint256 staker2Rewards = bPRIME.balanceOf(staker2);
    uint256 staker3Rewards = bPRIME.balanceOf(staker3);
    assertEq(staker1Rewards, staker2Rewards);
    assertEq(staker1Rewards, staker3Rewards);
}
```
This test will pass, proving that `staker1` that staked when the `rewardsExpiryThreshold`
was 24, lost their rewards for `epoch1` when `rewardsExpiryThreshold` was updated to 1.

## Impact
Users that have staked before the `rewardsExpiryThreshold` change, will lose out on
rewards since they do not expect this change.

Additionally this introduces a centralization risk, malicious or careless `tokenManager` can
cause the users to lose out on rewards.

## Recommendation
Keep track of what the `rewardsExpiryThreshold` was when that epoch happened, in
order to not make this variable backwards compatible. `rewardsExpiryThreshold` change
should not apply to epochs that have happened before the change.
