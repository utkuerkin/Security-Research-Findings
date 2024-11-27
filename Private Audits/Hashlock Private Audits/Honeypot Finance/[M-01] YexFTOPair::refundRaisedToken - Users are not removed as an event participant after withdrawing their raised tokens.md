## Description
Users are added as event participants when they deposit `raisedToken` into a pair.
However, when users withdraw their tokens, they are not removed as event
participants. This will lead to a situation where users who are no longer funding the pair
will benefit as an event participant.

## Vulnerability Details
The deposit function shown below, allows users to deposit `raisedToken` into a pair.
```solidity
function deposit(address raisedToken, address launchedToken, uint256 raisedTokenAmount) external override {
    _deposit(raisedToken, launchedToken, raisedTokenAmount);
}
```
This function will call the `_deposit` function which will call the `depositRaisedToken`
function in the `YexFTOPair` contract. Take a look at this function:
```solidity
function depositRaisedToken(address depositor, uint256 amount) external override whenNotPaused {
    //rest of the function
    /**
     * The addEvent function in the FTOFactory updates the storage variable
     * to reflect that the depositor has participated in this FTOPair.
     */
    IYexFTOFactory(factory).addEvent(depositor, address(this));
    emit DepositRaisedToken(depositor, amount);
}
```
However, when the pair is paused and this user withdraws their tokens with the
`refundRaisedToken` function shown below:
```solidity
function refundRaisedToken() external override lock whenPaused {
    // Verify that msg.sender is a valid address that had deposited RaisedToken.
    uint256 deposit_amount = raisedTokenDeposit[msg.sender];
    require(deposit_amount > 0, "refundable amount is 0");
    raisedTokenDeposit[msg.sender] = 0;
    depositedRaisedToken -= deposit_amount;
    IERC20(raisedToken).transfer(msg.sender, deposit_amount);
    emit Refund(msg.sender, deposit_amount);
}
```
It is observed that this user will not be removed as an event participant.

Take a look at the `addEvent` function in the `YexFTOFactory` contract below:
```solidity
function addEvent(address depositor, address ftoPair) external override {
    require(IYexFTOPair(ftoPair).raisedTokenDeposit(depositor) != 0, "Not participate in this rasing."); <@
    if (events_map[depositor][ftoPair] == false) {
        events_map[depositor][ftoPair] = true;
        eventParticipants[depositor].push(ftoPair);
    }
}
```
It is observed in the highlighted part that users who are not depositors should not be
event participants. This logic does not hold up when users withdraw their tokens.

## Impact
Users that withdraw their `raisedToken` from the pair will stay as event participants even
though they no longer have any deposited tokens in the pair.

## Recommendation
Remove users from the mappings and arrays that are relevant to the events when they
withdraw their tokens from the pair. As an example, implement a new function and an
event shown below and call this function in the `refundRaisedToken` function.
```solidity
// Event to be emitted when a user is removed from an event
event EventRemoved(address indexed depositor, address indexed ftoPair);

function removeEvent(address depositor, address ftoPair) external override {
    require(events_map[depositor][ftoPair] == true, "User is not a participant in this event.");
    // Remove the mapping
    events_map[depositor][ftoPair] = false;
    // Find and remove the ftoPair from the eventParticipants array
    uint256 length = eventParticipants[depositor].length;
    for (uint256 i = 0; i < length; i++) {
        if (eventParticipants[depositor][i] == ftoPair) {
            // Replace the found element with the last element
            eventParticipants[depositor][i] = eventParticipants[depositor][length - 1];
            // Remove the last element
            eventParticipants[depositor].pop();
            break;
        }
    }
    emit EventRemoved(depositor, ftoPair);
}
```
