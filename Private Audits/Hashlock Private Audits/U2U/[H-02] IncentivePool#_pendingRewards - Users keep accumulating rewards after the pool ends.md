## Description

The `_pendingRewards` function calculates a user’s rewards based on the amount of time that has passed since the last `_harvest` call, the number of tokens staked, and the rewards rate per second. However, this function does not take into account when the staking pool ends, causing users to accumulate more rewards after the pool ends.

## Vulnerability Details

The `_pendingRewards` function calculates a user’s rewards.

```solidity
function _pendingRewards(address _user) internal view returns (uint256) {
    unchecked {
        (uint256 totalStaked, uint256 latestHarvest, ) = _getUserInfo(
            _user
        );
        if (block.timestamp < startTime) return 0;
        if (latestHarvest < startTime) {
            latestHarvest = startTime;
        }
        if (totalStaked == 0) return 0;
        uint256 timeRewards = block.timestamp - latestHarvest;
        return timeRewards.mul(totalStaked).mul(_rewardsRatePerSecond());
    }
}
```

As observed in the highlighted part of the function, no checks were done on when the pool ended. This means that users will keep accumulating rewards after the pool ends.

This will cause discrepancies in the expected reward that will be paid to a user. Think of the scenario where 5 users have staked `100e6` tokens for the full pool duration. A user’s reward would be:

```
100e6 * 90 days * 857338 = 666666028800000000000
```

The protocol would be depositing approximately this amount times 5 to the contract in order to pay for users’ rewards. However, since the rewards will be accumulating after the pool ends, if some users do not claim their rewards right when the pool ends, there might be insufficient U2U in the contract to pay for all rewards.

Another problematic scenario is when any user does not claim their rewards when the pool ends, rewards will keep accumulating. For example, if this user claims 270 days after they have staked, they would be eligible for approximately `1999e18` U2U. There is a chance the contract will not have enough tokens to pay for their rewards, reverting any unstake or harvest call and causing not only their rewards but also their staked tokens to be stuck in the contract until enough U2U has been funded.

## Proof of Concept

An example of a test that simulates the vulnerability is below.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../IncentivePool.sol";
import "../../libs/TransferHelper.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

// Mock ERC20 token to represent pUSDT
contract MockERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {}

    function decimals() public view virtual override returns (uint8) {
        return 6;
    }

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract IncentivePoolTest is Test {
    IncentivePool public incentivePool;
    MockERC20 public pUSDT;

    address user = address(0x1);
    address admin = address(0x2);
    address user2 = address(0x3);

    uint256 constant INITIAL_PUSDT_SUPPLY = 1_000_000 * 1e6;
    uint256 constant START_TIME = 1730000000;
    uint256 constant END_TIME = START_TIME + 90 days;
    uint256 constant STAKE_AMOUNT = 1000 * 1e6;
    uint256 constant MIN_STAKE_AMOUNT = 10 * 1e6;
    uint256 constant MAX_USDT_POOL_CAP = 1_500_000 * 1e6;
    uint256 constant MAX_STAKE_AMOUNT = 10000 * 1e6;

    function setUp() public {
        // Deploy Mock pUSDT token and mint to users
        pUSDT = new MockERC20("Mock pUSDT", "pUSDT");
        pUSDT.mint(user, 10e6);
        pUSDT.mint(user2, 20e6);

        // Deploy IncentivePool contract
        incentivePool = new IncentivePool(address(pUSDT), START_TIME);

        // Assign DEFAULT_ADMIN_ROLE to admin
        incentivePool.grantRole(incentivePool.DEFAULT_ADMIN_ROLE(), admin);

        // Set ignoreSigner to true to bypass signature checks for testing
        vm.prank(admin);
        incentivePool.setIgnoreSigner(true);

        // Set claimableTime to allow harvesting
        vm.prank(admin);
        incentivePool.setClaimableTime(START_TIME + 1);

        // Approve the maximum allowance for IncentivePool to spend pUSDT on behalf of the users
        vm.prank(user);
        pUSDT.approve(address(incentivePool), type(uint256).max);
        vm.prank(user2);
        pUSDT.approve(address(incentivePool), type(uint256).max);

        // Fund the IncentivePool contract with U2U (native asset) for rewards
        vm.deal(address(incentivePool), 10000 ether);
    }

    function testUserKeepsAccumulatingRewards() public {
        // Forward time to after start time
        vm.warp(START_TIME);

        // Stake the amount
        vm.prank(user);
        incentivePool.stake(10e6, END_TIME, "");
        vm.prank(user2);
        incentivePool.stake(10e6, END_TIME, "");

        // Forward time to when pool ends and user unstakes
        vm.warp(END_TIME + 1);
        vm.prank(user);
        incentivePool.unstake();

        // Forward time 10000000 seconds and user2 unstakes
        vm.warp(END_TIME + 10000001);
        vm.prank(user2);
        incentivePool.unstake();

        // Assertion and logs
        assertGt(user2.balance, user.balance);
        console.log("user1 bal:", user.balance);
        console.log("user2 bal:", user2.balance);
    }
}
```

## Impact

Users will accumulate more rewards than expected. This will cause the user’s rewards and staked tokens to be stuck in the contract.

## Recommendation

Change the reward calculation logic to consider when the pool has ended. An example is shown below.

```solidity
function _pendingRewards(address _user) internal view returns (uint256) {
    unchecked {
        (uint256 totalStaked, uint256 latestHarvest, ) = _getUserInfo(_user);
        
        if (block.timestamp < startTime) return 0;
        if (latestHarvest < startTime) {
            latestHarvest = startTime;
        }
        
        if (totalStaked == 0) return 0;

        uint256 endCalculationTime = block.timestamp < endTime ? block.timestamp : endTime;
        
        uint256 timeRewards = endCalculationTime - latestHarvest;
        return timeRewards.mul(totalStaked).mul(_rewardsRatePerSecond());
    }
}
```

This modification ensures that rewards are only calculated up until the pool’s end time, preventing users from accumulating additional rewards after the pool has ended.
