## Summary

Miscalculation of rewards causes funds to be permanently stuck in the contract.

## Vulnerability Details

Documents state that “This contract allows the creator to invite a select group of people to vote on something and provides an eth reward to the `for` voters if the proposal passes.”. However in the contract, when a vote passes, rewards are calculated with this integer `uint256 rewardPerVoter = totalRewards / totalVotes;` ([Line #192](https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L192)). `totalVotes` is calculated with this integer `uint256 totalVotes = totalVotesFor + totalVotesAgainst;` ([Line #172](https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L172)**).** This means that the rewards are not shared according to total number of ‘for’ voters.

`uint256 rewardPerVoter = totalRewards / totalVotes;` and `rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);` [lines](https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L207C16-L207C16) of code causes funds to get permanently stuck in the contract if at least one voter votes ‘against’.

For example when vote is passed with 2 ‘for’ voters and 1 ‘against’ voter 1 ETH reward would be split split into 3 due to use dividing with `totalVotes` this causes ‘for’ voters to get 0.3333… ETH each while rest of the ETH is stuck in the contract.

## PoC

Add `import "forge-std/console.sol";` into imports in `VotingBoothTest.t.sol`

Add the following test in `VotingBoothTest.t.sol`

```jsx
//Check if there are funds left in the contract after vote passes when one voter votes against.
function testVotePassesAndSomeMoneyIsStuckInTheContract() public {
        vm.prank(address(0x1));
        booth.vote(true);

        vm.prank(address(0x2));
        booth.vote(true);

        vm.prank(address(0x3));
        booth.vote(false);
				
				console.log(address(0x1).balance);
        console.log(address(0x2).balance); 
        console.log(address(0x3).balance);
        console.log(address(booth).balance);

        assert(!booth.isActive() && address(booth).balance > 0);
    }
```

Run `forge test --match-test testVotePassesAndMoneyIsStuckInTheContract -vvvv` in foundry.

Console prints the following lines and proving that after voting ends and rewards are sent to ‘for’ voters, 0.33333 ETH is still stuck in the contract with no way of recovering it.

```jsx
 		console::log(3333333333333333333 [3.333e18]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log(3333333333333333334 [3.333e18]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log(0) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log(3333333333333333333 [3.333e18]) [staticcall]
    │   └─ ← ()
```

## Impact

Every voting that passes that has ‘against’ votes, rewards that ‘for’ voters receive will be less than the total amount that was funded into the contract. With no other way of recovering funds, rest of the rewards will be permanently stuck in the contract.

## Tools Used

Foundry and manual review.

## Recommendations

I recommend two possible fixes for this vulnerability.

### Recommendation #1
If all the funds are meant to be split equally between ‘for’ voters update the following [lines](https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L191-L211) in `_distributeRewards()` as shown below:

```jsx
else {
            uint256 rewardPerVoter = totalRewards / totalVotesFor;

            for (uint256 i; i < totalVotesFor; ++i) {
                // proposal creator is trusted when creating allowed list of voters,
                // findings related to gas griefing attacks or sending eth
                // to an address reverting thereby stopping the reward payouts are
                // invalid. Yes pull is to be preferred to push but this
                // has not been implemented in this simplified version to
                // reduce complexity & help you focus on finding the
                // harder to find bug

                // if at the last voter round up to avoid leaving dust; this means that
                // the last voter can get 1 wei more than the rest - this is not
                // a valid finding, it is simply how we deal with imperfect division
                if (i == totalVotesFor - 1) {
                    rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotesFor, Math.Rounding.Ceil);
                }
                _sendEth(s_votersFor[i], rewardPerVoter);
            }
        }
```
### PoC for Recommendation #1:
<details>
To prove this fix works, run `forge test --match-test testVotePassesAndMoneyIsSent -vvvv` in foundry:

```jsx
function testVotePassesAndMoneyIsSent() public {
        vm.prank(address(0x1));
        booth.vote(true);

        vm.prank(address(0x2));
        booth.vote(true);

        vm.prank(address(0x3));
        booth.vote(false);

        console.log(address(0x1).balance);
        console.log(address(0x2).balance); 
        console.log(address(0x3).balance);
        console.log(address(booth).balance);

        assert(!booth.isActive() && address(booth).balance == 0);
    }
```

foundry prints the following lines:

```jsx
    ├─ [0] console::log(5000000000000000000 [5e18]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log(5000000000000000000 [5e18]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log(0) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log(0) [staticcall]
    │   └─ ← ()
```

Test proves that with this fix, rewards are split between ‘for’ voters and funds do not get stuck in the contract.
</details>

### Recommendation #2
If the intended implementation is that rewards of ‘for’ voters should be divided based on the total amount of votes, update the following [lines](https://github.com/Cyfrin/2023-12-Voting-Booth/blob/a3241b1c530529a4f739f9462f720e8561ad45ca/src/VotingBooth.sol#L191-L211) in `_distributeRewards()` as shown below:

```jsx
else {
            uint256 rewardPerVoter = totalRewards / totalVotes;

            for (uint256 i; i < totalVotesFor; ++i) {
                // proposal creator is trusted when creating allowed list of voters,
                // findings related to gas griefing attacks or sending eth
                // to an address reverting thereby stopping the reward payouts are
                // invalid. Yes pull is to be preferred to push but this
                // has not been implemented in this simplified version to
                // reduce complexity & help you focus on finding the
                // harder to find bug

                // if at the last voter round up to avoid leaving dust; this means that
                // the last voter can get 1 wei more than the rest - this is not
                // a valid finding, it is simply how we deal with imperfect division
                if (i == totalVotesFor - 1) {
                    rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
                }
                _sendEth(s_votersFor[i], rewardPerVoter);
            
            }
        _sendEth(s_creator, address(this).balance);
```

`_sendEth(s_creator, address(this).balance);` line that is added will make sure that funds that are left after rewards are distributed will be sent back to the creator.

### PoC for Recommendation #2: 
<details>
To prove this fix works, run `forge test --match-test testVotePassesAndMoneyIsSent -vvvv` in foundry.

```jsx
function testVotePassesAndMoneyIsSent() public {
        vm.prank(address(0x1));
        booth.vote(true);

        vm.prank(address(0x2));
        booth.vote(true);

        vm.prank(address(0x3));
        booth.vote(false);

        console.log(address(0x1).balance);
        console.log(address(0x2).balance); 
        console.log(address(0x3).balance);
        console.log(address(this).balance);
        console.log(address(booth).balance);

        assert(!booth.isActive() && address(booth).balance == 0);
    }
```

Running this test prints the following lines:

```jsx
    ├─ [0] console::log(3333333333333333333 [3.333e18]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log(3333333333333333334 [3.333e18]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log(0) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log(3333333333333333333 [3.333e18]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log(0) [staticcall]
    │   └─ ← ()
```

This proves that rewards are split between ‘for’ voters and remaining funds are sent to the creator.
</details>
		
### Result

Pending