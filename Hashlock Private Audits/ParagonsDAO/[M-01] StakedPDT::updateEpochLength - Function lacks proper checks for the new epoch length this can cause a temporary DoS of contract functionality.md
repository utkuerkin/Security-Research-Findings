## Description
The `updateEpochLength` function allows the privileged `EPOCH_MANAGER` role to
change the length of epochs. Lack of checks in this function can lead to temporary DoS,
making it impossible to move to the next epoch until the length is updated again.

## Vulnerability Details
The `updateEpochLength` function shown below updated the length of epochs.
```solidity
function updateEpochLength(uint256 newEpochLength) external onlyRole(EPOCH_MANAGER) {
    require(newEpochLength > 0, "Invalid new epoch length");
    uint256 previousEpochLength = epochLength;
    epochLength = newEpochLength;
    epoch[currentEpochId].endTime = epoch[currentEpochId].startTime + newEpochLength;
    emit UpdateEpochLength(currentEpochId, previousEpochLength, newEpochLength);
}
```
As seen in this function, there are no checks done to make sure that the new epoch end
time is greater than `contractLastInteracted`.

If the new `epoch.endTime` is less than the `contractLastInteracted`, when the `distribute`
function is called, this function will make a call to the `contractWeight` function which
will call the `_weightIncreaseSinceInteraction` function to calculate contract weight.
```solidity
function distribute() external onlyRole(EPOCH_MANAGER) {
    uint256 _currentEpochId = currentEpochId;
    Epoch memory _currentEpoch = epoch[_currentEpochId];
    if (block.timestamp >= _currentEpoch.endTime) epoch[_currentEpochId].weightAtEnd = contractWeight();
    //rest of the function
}

function contractWeight() public view returns (uint256 contractWeight_) {
    Epoch memory _epoch = epoch[currentEpochId];
    uint256 _weightIncrease = _weightIncreaseSinceInteraction(
        Math.min(block.timestamp, _epoch.endTime), Math.max(contractLastInteraction, _epoch.startTime), totalSupply()
    );
    contractWeight_ = _weightIncrease + _contractWeight;
}

function _weightIncreaseSinceInteraction(uint256 timestamp, uint256 lastInteraction, uint256 baseAmount)
    internal
    pure
    returns (uint256 additionalWeight_)
{
    uint256 _timePassed = timestamp - lastInteraction;
    uint256 _multiplierReceived = (1e18 * _timePassed) / 1 days;
    additionalWeight_ = (baseAmount * _multiplierReceived) / 1e18;
}
```
However, since the `timestamp` parameter in this function is less than the `lastInteraction`,
this call will revert due to underflow.

## Proof of Concept
Implement and run the following test in the `StakedPDT_claim.t.sol` file.
```solidity
function test_cannotMoveToNextEpoch() public {
    uint256 POOL_SIZE = 1e18;
    uint256 initialBalance = 1e18;
    bPDTOFT.mint(staker1, initialBalance * 2);
    /// EPOCH 0
    _creditPRIMERewardPool(POOL_SIZE);
    _moveToNextEpoch(0);
    // stake in epoch 1
    (uint256 epochStartTime, uint256 epochEndTime,) = bStakedPDT.epoch(1);
    vm.warp(epochStartTime);
    // advance time
    vm.warp(epochStartTime + 2 weeks);
    //user stakes
    vm.startPrank(staker1);
    bPDTOFT.approve(bStakedPDTAddress, initialBalance);
    bStakedPDT.stake(staker1, initialBalance);
    vm.stopPrank();
    //update epoch lenght
    vm.startPrank(epochManager);
    bStakedPDT.updateEpochLength(1 weeks);
    vm.stopPrank();
    //move to next epoch
    _creditPRIMERewardPool(POOL_SIZE);
    _moveToNextEpoch(1);
}
```
This test will fail due to an underflow error when the `distribute` function is called.

## Impact
It will be impossible to move on to the next epoch until epoch length is updated again,
causing a DoS of the contract.

## Recommendation
Implement a check in the `updateEpochLength` function to make sure that the new
`epoch.endTime` is greater than the `contractLastInteraction`.
