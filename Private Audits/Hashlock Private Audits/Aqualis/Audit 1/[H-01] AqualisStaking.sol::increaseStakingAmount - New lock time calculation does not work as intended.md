## Description
Users with an active stake can call `increaseStakingAmount` function in order to
increase their staked amount. Based on this staked amount, compared to the user's
already existing stake, the function should increase the lock time of the user's stake.
However, the calculation for the lock time does not work as intended.

## Vulnerability Details
The `increaseStakingAmount` function updates the `newLockTime` which is used to
calculate the new lock time of users staked tokens.
```solidity
function increaseStakingAmount(uint256 _amount) external {
    require(_amount > 0, "Amount must be > 0");
    address staker = _msgSender();
    Stakes memory stake = stakers[staker];
    require(stake.amount > 0, "Stake does not exist");
    aqualisToken.transferFrom(staker, address(this), _amount);
    /// @dev increment stake amount
    stake.amount += _amount;
    uint32 remainingLockTime = stake.unlockAt - uint32(block.timestamp);
    uint32 initialStakeTime = uint32(stake.weeksNum) * 1 weeks;
    /// @dev new Lock Time = (initialStakeTime - remainingLockTime) * portion of additional amount +
    /// remainingLockTime
    uint32 newLockTime = uint32(
        (initialStakeTime - remainingLockTime) * (_amount / stake.amount)
    ) + remainingLockTime;
    /// @dev rounded-up to weeks
    uint8 newLockTimeInWeeks = uint8(newLockTime / 1 weeks + 1);
    stake.unlockAt =
        uint32(block.timestamp) +
        uint32(newLockTimeInWeeks * 1 weeks);
    stake.weeksNum = newLockTimeInWeeks;
    stake.aqualisPower = _calculateAP(stake.amount, newLockTimeInWeeks);
    stakers[staker] = stake;
    _updateReward(staker, stake.aqualisPower);
    emit StakeUpdated(staker, stake.amount, stake.aqualisPower, stake.unlockAt);
}
```
Since the function increments the added staked amount to the already existing stakes
before doing the calculation, the following calculation will always return the
`remainingLockTime`.
```solidity
uint32 newLockTime = uint32(
        (initialStakeTime - remainingLockTime) * (_amount / stake.amount)
    ) + remainingLockTime;
    uint8 newLockTimeInWeeks = uint8(newLockTime / 1 weeks + 1);
```
This is caused by the way solidity handles divisions and decimal numbers. Since
`stake.amount` is greater than `_amount`, their division always returns 0. Then the
function calculates the `newLockTimeInWeeks` by adding 1 to `newLockTime`. This
results in the function always increasing the `remainingLockTime` by 1 week.

## Proof of Concept
Add the following test in `AqualisStaking.t.sol` and run it.
```solidity
function test_newLockTimeIsRemaningLockTimePlusOne() external {
    _createPrankStake(staker1, 100, 30);
    skip(5 * 1 weeks);
    uint256 oldLockTimeInWeeks = staking.getRemainingLockTimeInWeeks(staker1);
    vm.startPrank(staker1);
    token.approve(address(staking), 900 * 1e18);
    staking.increaseStakingAmount(900 * 1e18);
    vm.stopPrank();
    uint256 newLockTimeInWeeks = staking.getRemainingLockTimeInWeeks(staker1);
    (uint256 newStake, uint128 newAP, uint32 newUnlockAt, ) = staking
        .getStakeInfo(staker1);
    assertEq(oldLockTimeInWeeks + 1, newLockTimeInWeeks);
}
```
## Impact
Function does not work as intended. Users that increase their staking amount can
inate their `AP` without having to lock their tokens for an extended amount of time.

## Recommendation
Use `ABDKMath64x64` for decimals precision or change the calculations to avoid the division
returning 0.



