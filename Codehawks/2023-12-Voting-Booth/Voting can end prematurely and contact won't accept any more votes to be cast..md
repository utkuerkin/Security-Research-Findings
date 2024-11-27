## Summary

Incorrect calculation leads to premature voting termination, causing the contract to reject additional votes even if `allowListLength` exceeds the number of votes cast.

## Vulnerability Details

`if (totalCurrentVotes * 100 / s_totalAllowedVoters >= MIN_QUORUM)` [line](https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L156C9-L156C76) of code in `vote()` function causes voting to end early since it checks for `totalCurrentVotes` instead of checking `for` and `against` votes separately.

For example when there are 5 total voters, as soon as 3 votes are cast `if (totalCurrentVotes * 100 / s_totalAllowedVoters >= MIN_QUORUM)` will be `if(true)` because `3*100/5 = 60` which is `>= MIN_QUORUM` (`MIN_QUORUM` is set to 51).

In short, when `totalCurrentVotes` / `s_totalAllowedVoters` ≥ 0.51 condition is met voting will end. Most dangerous cases are when there are 3 or 7 allowed voters. In these cases voting can end prematurely and unsuccessfully even if number `for` votes is equal to number of `against` votes.

## PoC

```jsx
function testVotingDoesNotEndEarly() public {
        vm.prank(address(0x1));
        booth.vote(true);
        console.log(booth.isActive());

        vm.prank(address(0x2));
        booth.vote(false);
        console.log(booth.isActive());

        vm.prank(address(0x3));
        booth.vote(false);
        console.log(booth.isActive());

        vm.prank(address(0x4));
        booth.vote(true);
        console.log(booth.isActive());

        vm.prank(address(0x5));
        booth.vote(false);
        console.log(booth.isActive());

    }
```

Write the test shown above in `VotingBoothTest.t.sol` and run it with `forge test --match-test testVotingEndsEarly -vvvv` in the terminal. Test will revert and Foundry will display the following results in the terminal.

```jsx
[171714] VotingBoothTest::testVotingEndsEarly()
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000001)
    │   └─ ← ()
    ├─ [76513] VotingBooth::vote(true)
    │   └─ ← ()
    ├─ [292] VotingBooth::isActive() [staticcall]
    │   └─ ← true
    ├─ [0] console::log(true) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000002)
    │   └─ ← ()
    ├─ [80143] VotingBooth::vote(false)
    │   ├─ [55] VotingBoothTest::receive{value: 10000000000000000000}()
    │   │   └─ ← ()
    │   └─ ← ()
    ├─ [292] VotingBooth::isActive() [staticcall]
    │   └─ ← false
    ├─ [0] console::log(false) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000003)
    │   └─ ← ()
    ├─ [523] VotingBooth::vote(false)
    │   └─ ← revert: DP: voting has been completed on this proposal
    └─ ← revert: DP: voting has been completed on this proposal
```

`console::log` shows that when `address(0x3)` casts their vote `booth.isActive()` becomes false and subsequent votes are rejected with `revert: DP: voting has been completed on this proposal`

## Impact

This vulnerability causes voting to end before all allowed voters have casted their votes. In most dangerous cases voting will fail while number of `for` votes is equal to number of `against` votes.

## Tools Used

Foundry and manual review.

## Recommendations

Add the lines shown below in our contract under `uint256 private s_totalCurrentVotes;` ([L #67](https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L67C4-L67C41))

```jsx
    //total number of current for votes
    uint256 private s_totalCurrentVotesFor;

    //total number of current against votes
    uint256 private s_totalCurrentVotesAgainst;
```

Then update `vote()` function as shown below.

```jsx
function vote(bool voteInput) external {
        // prevent voting if already completed
        require(isActive(), "DP: voting has been completed on this proposal");

        // current voter
        address voter = msg.sender;

        // prevent voting if not allowed or already voted
        require(s_voters[voter] == ALLOWED, "DP: voter not allowed or already voted");

        // update storage to record that this user has voted
        s_voters[voter] = VOTED;

        // add user to either the `for` or `against` list
         if (voteInput) {
            s_votersFor.push(voter);
            s_totalCurrentVotesFor++;
        } else {
            s_votersAgainst.push(voter);
            s_totalCurrentVotesAgainst++;
        }
        // check if quorum has been reached. Quorum is reached
        // when at least 51% of the total allowed voters have cast
        // their vote. For example if there are 5 allowed voters:
        //
        // first votes For
        // second votes For
        // third votes Against
        //
        // Quorum has now been reached (3/5) and the vote will pass as
        // votesFor (2) > votesAgainst (1).
        //
        // This system of voting doesn't require a strict majority to
        // pass the proposal (it didn't require 3 For votes), it just
        // requires the quorum to be reached (enough people to vote)
        //
        if (s_totalCurrentVotesFor * 100 / s_totalAllowedVoters >= MIN_QUORUM || s_totalCurrentVotesAgainst * 100 / s_totalAllowedVoters >= MIN_QUORUM) {
            // mark voting as having been completed
            s_votingComplete = true;

            // distribute the voting rewards
            _distributeRewards();
        }
    }
```

Updated lines in `vote()` function are [lines #138-139](https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L138-L139) and [line #156](https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L156).

### PoC of Fix
<details>

```jsx
function testVotingDoesNotEndEarly() public {
        vm.prank(address(0x1));
        booth.vote(true);
        console.log(booth.isActive());

        vm.prank(address(0x2));
        booth.vote(false);
        console.log(booth.isActive());

        vm.prank(address(0x3));
        booth.vote(true);
        console.log(booth.isActive());

        vm.prank(address(0x4));
        booth.vote(false);
        console.log(booth.isActive());

        vm.prank(address(0x5));
        booth.vote(true);
        console.log(booth.isActive());
        
        assert(!booth.isActive());

    }
```

Write the test shown above in `VotingBoothTest.t.sol` and run it with `forge test --match-test testVotingDoesNotEndEarly -vvvv` in the terminal. The test will run successfully, and Foundry will display the following results in the terminal.
```jsx
[379554] VotingBoothTest::testVotingDoesNotEndEarly()
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000001)
    │   └─ ← ()
    ├─ [78996] VotingBooth::vote(true)
    │   └─ ← ()
    ├─ [292] VotingBooth::isActive() [staticcall]
    │   └─ ← true
    ├─ [0] console::log(true) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000002)
    │   └─ ← ()
    ├─ [70986] VotingBooth::vote(false)
    │   └─ ← ()
    ├─ [292] VotingBooth::isActive() [staticcall]
    │   └─ ← true
    ├─ [0] console::log(true) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000003)
    │   └─ ← ()
    ├─ [29196] VotingBooth::vote(true)
    │   └─ ← ()
    ├─ [292] VotingBooth::isActive() [staticcall]
    │   └─ ← true
    ├─ [0] console::log(true) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000004)
    │   └─ ← ()
    ├─ [29186] VotingBooth::vote(false)
    │   └─ ← ()
    ├─ [292] VotingBooth::isActive() [staticcall]
    │   └─ ← true
    ├─ [0] console::log(true) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::prank(0x0000000000000000000000000000000000000005)
    │   └─ ← ()
    ├─ [150757] VotingBooth::vote(true)
    │   ├─ [3000] 0x0000000000000000000000000000000000000001::fallback{value: 3333333333333333333}()
    │   │   └─ ← ()
    │   ├─ [600] PRECOMPILES::ripemd{value: 3333333333333333333}(0x)
    │   │   └─ ← 0x0000000000000000000000009c1185a5c5e9fc54612808977ee8f548b2258d31
    │   ├─ [200] 0x0000000000000000000000000000000000000005::fallback{value: 3333333333333333334}()
    │   │   └─ ← ()
    │   └─ ← ()
    ├─ [292] VotingBooth::isActive() [staticcall]
    │   └─ ← false
    ├─ [0] console::log(false) [staticcall]
    │   └─ ← ()
    ├─ [292] VotingBooth::isActive() [staticcall]
    │   └─ ← false
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.24ms
```

`console::log` shows that `isActive()` only turns false when all 5 voters have cast their votes. This test proves that the recommended fix works.
</details>

### Result
Pending
