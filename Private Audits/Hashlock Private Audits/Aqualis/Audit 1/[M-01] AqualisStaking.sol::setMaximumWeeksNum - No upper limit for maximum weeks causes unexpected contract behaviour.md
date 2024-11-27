## Description
The `setMaximumWeeksNum` function allows the owner to change the maximum number
of weeks users can stake for. This function needs additional checks.

## Vulnerability Details
The `setMaximumWeeksNum` function not requiring an upper limit can cause the calls to
the `unstakeWithPenalty` function to revert due to underflow.
```solidity
function setMaximumWeeksNum(uint8 _maximumWeeksNum) external onlyOwner {
    if (_maximumWeeksNum == 0x0) revert ZeroAmount();
    maximumWeeksNum = _maximumWeeksNum;
}
```
As shown above, this function updates the `maximumWeeksNum` variable. This variable
used to set an upper limit on the amount on how long the users can stake for.

In an example scenario where the `maximumWeeksNum` variable is set to 150, and the
`penaltyPerWeek` variable set to 10 (This can be set as high as 999). Users that
unstake their tokens with penalty while their `remainingLockTimeInWeek` is more
than 100 will cause the calls to the `unstakeWithPenalty` function to revert due to
underflow.
```solidity
function unstakeWithPenalty(uint256 _amount) external nonReentrant {
    //rest of the function...
    uint256 remainingWeeks = getRemainingLockTimeInWeeks(staker);
    require(remainingWeeks > 0, "Stake Matured");
    /// @dev 1% + penaltyPerWeek * remaining weeks (using PERCENT_FRACTION)
    uint256 penaltyPercent = 10 + penaltyPerWeek * remainingWeeks;
    /// @dev penalty divided by 1e3 (PERCENT_FRACTION denominator)
    uint256 penalty = (_amount * penaltyPercent) / 1000;
    uint256 returnAmount = _amount - penalty;
    //rest of the function...
}
```
This function in the given scenario will calculate the `penaltyPercent` as `10 + 10 *100 = 1010`. 
Then the function calculates the penalty as `(_amount * 1010) / 1000`. 
This results in the following condition: `penalty > _amount`.

When the function tries to calculate the returnAmount, this will revert due to
underflow.

## Proof of Concept
Add the following test in `AqualisStaking.t.sol` and run it.
```solidity
function test_MaxWeekCanBreakContract() external {
    vm.prank(owner);
    staking.setPenaltyPerWeek(10);
    _createPrankStake(staker1, 100, 150);
    skip(5 * 1 weeks);
    vm.expectRevert();
    vm.prank(staker1);
    staking.unstakeWithPenalty(100);
}
```

## Impact
Unchecked `maximumWeeksNum` amount will break the `unstakeWithPenalty` function.

## Recommendation
Add an upper limit to the `maximumWeeksNum` with the changes to the function as
shown below:
```solidity
function setMaximumWeeksNum(uint8 _maximumWeeksNum) external onlyOwner {
    if (_maximumWeeksNum == 0x0) revert ZeroAmount();
    require(_maximumWeeksNum * penaltyPerWeek <= 990);
    maximumWeeksNum = _maximumWeeksNum;
}
```
