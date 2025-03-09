## Description
The `addEvent` function is called by a pair contract when a user deposits `raisedToken` into
that pair. This function lacks proper checks to make sure that the pair inputted into this
function is a valid pair.

## Vulnerability Details
The `addEvent` function shown below, manages the events when a user deposits into a
pair contract.
```solidity
function addEvent(address depositor, address ftoPair) external override {
    require(IYexFTOPair(ftoPair).raisedTokenDeposit(depositor) != 0, "Not participate in this rasing.");
    if (events_map[depositor][ftoPair] == false) {
        events_map[depositor][ftoPair] = true;
        eventParticipants[depositor].push(ftoPair);
    }
}
```
As observed in the highlighted part, the only check done is checking the
`raisedTokenDeposit` amount of the depositor in that pair. This means that any user can
deploy a fake pair contract that will be compatible with the `IYexFTOPair` interface and
set their `raisedTokenDeposit` amount in this contract to be greater than 0.
These users can now call the `addEvent` function with their deployed fake pair contract
as the parameter and become an event participant for that fake pair.
```solidity
function events(address depositor) external view override returns (address[] memory pairs) {
    return eventParticipants[depositor];
}
```
As observed in the `events` function above this function will return all the fake pair
contracts as if the user has participated in actual events. Depending on the off-chain
use of events, users can manipulate statistics and leaderboards.

## Impact
Users can manipulate the `events` mappings and arrays as they wish. This can lead to
off-chain systems using `events` reading incorrect data.

## Recommendation
Implement proper checks to make sure that `events` can only be added by legitimate pair
contracts. This can be done by implementing a new modier that checks for a mapping.
Pairs launched through the `YexFTOFactory` contract should be added to this mapping
and this modifier should act as access control for `addEvent` function.

An alternative change could be to change the `addEvent` function to receive two
parameters: `raisedToken` and `launchedToken`. With these parameters, the function can
use these values to get the pair corresponding to these tokens.
