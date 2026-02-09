**Official Finding:** [Sherlock Submission](https://github.com/sherlock-audit/2023-11-covalent-judging/issues/103#issue-2102539419)

## Delegators can cause loss of rewards to validators.

## Summary

Delegator is able to grief a Validator and cause the Validator considerable loss of rewards.

## Vulnerability Detail

> there is an offchain system, which controls the behaviour of `rewardValidators` and the "amounts" that is passed to it:
> 1. when there is a Quorum event emitted, the offchain system takes note of that, and queries for the current staked amount of the system.
> 2. then the system determines the how much cqt each validator deserve based on ratio of stakes and per-blockspecimen-reward.
> 3. the values are tracked. Then every 12 hours, the reward values are aggregated and `rewardValidators` is called.

Shown above is the intended use of the system. A malicious user will stake enough on a validator to fill `delegationMaxCap`, and frontrun all `_finalizeWithParticipants()` (this function emits the Quorum event) calls with their own `_unstake()` call to lower the amount of reward this validator would get. Malicious delegator will call backrun this same call with their `recoverUnstaking()` call and fill out `delegationMaxCap` again.

## Impact

Loss of funds for validators

## Code Snippet

## Tool used

Manual review

## Recommendation

Implement a new cooldown period for Unstakings and use it in `recoverUnstaking()` function. Example shown below.

```solidity
struct Unstaking {
    uint128 coolDownEnd; // epoch when unstaking can be redeemed
++  uint128 recoverCooldownEnd; // epoch when recovering can happen 
    uint128 amount; // # of unstaked CQT
}

function recoverUnstaking(uint128 amount, uint128 validatorId, uint128 unstakingId) external whenNotPaused {
    require(validatorId < validatorsN, "Invalid validator");
    require(_validators[validatorId].unstakings[msg.sender].length > unstakingId, "Unstaking does not exist");
++  require(uint128(block.number) > us.recoverCoolDownEnd);
    //... rest of the function ...
}
```
