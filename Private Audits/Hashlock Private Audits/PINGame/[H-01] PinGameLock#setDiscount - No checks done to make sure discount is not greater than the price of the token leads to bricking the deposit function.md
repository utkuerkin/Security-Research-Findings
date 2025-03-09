## Description
The `setDiscount` function allows the owner to add a discount to the discount arrays of a token. The `deposit` function has calculations for the `actualPrice` of the token by subtracting the discount from the price of this token. As there are no checks done to make sure that the discount cannot exceed the price of a token, if the discount amount is set higher than the price, all calls to the `deposit` function will revert due to underflow. This vulnerability also breaks the `prices` view function.

## Vulnerability Details
The `setDiscount` function allows the owner to add a discount to the discount arrays of a token.

```solidity
function setDiscount(address _token, uint256 discount_price, uint256 discount_amount) onlyOwner external {
    require(discount_price > 0, "price should be greater than 0");
    require(discount_amount >= 0, "amount should be greater than 0");
    require(supportedTokens[_token], "token not exist");
    
    // sort the discounts array in descending order
    discountsAmounts[_token].push(discount_price);
    uint256[] memory sortedAmounts = Arrays.sort(discountsAmounts[_token], Comparators.gt);
    discountsAmounts[_token] = sortedAmounts;
    discounts[_token][discount_price] = discount_amount;
    updateCurrentDiscount(_token);
    
    emit SetDiscount(_token, discount_price, discount_amount);
}
```

This function has checks to ensure that the discount price and discount amounts are greater than 0 and that the token is supported. The function then adds a new discount to the token’s discount array and sorts this array to make the highest discount the first item. However, there are no checks to ensure that the added discount is less than the price of the token.

In the `deposit` function, the `actualPrice` of the token is calculated by subtracting the discount from the price of this token.

```solidity
function deposit(address _token, uint256 amount, string memory refererCode) external {
    require(supportedTokens[_token], "token not supported");
    require(amount > 0, "amount should be greater than 0");
    require(originPrices[_token] > 0, "price should be greater than 0");
    
    uint256 cd = getCurrentDiscount(_token);
    uint256 actualPrice = originPrices[_token] - cd;
    
    // rest of the function
}
```

If this token’s discount is higher than its price, this calculation will cause a revert due to an underflow. As there are no ways to remove discounts from the discount array, the only fix to this problem is updating the price of the token to be higher than the discount. However, this fix is a workaround that breaks the integrity of the contract. This vulnerability also breaks the `prices` view function.

## Proof of Concept
1) Set up a Foundry test suite

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../contracts/PinGameLock.sol";
import "../contracts/MyToken.sol";

contract PinGameLockTest is Test {
    PinGameLock public lock;
    MyToken public token;
    address owner = address(1);
    address user = address(2);
    address user2 = address(3);

    function setUp() public {
        // Deploy test token
        token = new MyToken("Test Token", "TEST");
        token.mint(user, 1000 ether);
        token.mint(user2, 1000 ether);

        // Deploy PinGameLock
        lock = new PinGameLock();
        address[] memory tokens = new address[](1);
        tokens[0] = address(token);
        uint256[] memory prices = new uint256[](1);
        prices[0] = 100 ether;

        // Initialize with owner and supported tokens
        lock.initialize(owner, prices, tokens);

        // Set discount valid time
        vm.prank(owner);
        lock.setDiscountValidTo(block.timestamp + 10 days);

        // Approve tokens
        vm.prank(user);
        token.approve(address(lock), type(uint256).max);

        vm.prank(user2);
        token.approve(address(lock), type(uint256).max);
    }
```

2) Implement and run the following test to observe the vulnerability.

```solidity
function testUnderflowDiscountUpdate() public {
    vm.startPrank(owner);
    // Setting discount higher than price - this should succeed
    lock.setDiscount(address(token), 110 ether, 1);
    vm.stopPrank();

    vm.startPrank(user);
    // Now try to deposit - this should revert
    vm.expectRevert(stdError.arithmeticError);
    lock.deposit(address(token), 100 ether, "TEST");
    vm.stopPrank();

    vm.startPrank(owner);
    // Setting discount lower does not fix
    lock.setDiscount(address(token), 20 ether, 1);
    vm.stopPrank();

    vm.startPrank(owner);
    // prices function is also broken
    vm.expectRevert(stdError.arithmeticError);
    uint256 price = lock.prices(address(token));
    vm.assertEq(price, 0);
    vm.stopPrank();

    vm.startPrank(user);
    // Now try to deposit again - this should revert
    vm.expectRevert(stdError.arithmeticError);
    lock.deposit(address(token), 100 ether, "TEST");
    vm.stopPrank();
}
```

## Impact
The `deposit` function, which is integral to the contract's functionality, will be permanently bricked and will not work.

## Recommendation
Implement checks in the `setDiscount` function to ensure that the discount cannot be greater than the price of a token.

```solidity
function setDiscount(address _token, uint256 discount_price, uint256 discount_amount) onlyOwner external {
    require(discount_price > 0, "price should be greater than 0");
    require(discount_amount >= 0, "amount should be greater than 0");
    require(supportedTokens[_token], "token not exist");
    require(discount_price <= originPrices[_token], "discount cannot exceed price");
    
    // sort the discounts array in descending order
    discountsAmounts[_token].push(discount_price);
    uint256[] memory sortedAmounts = Arrays.sort(discountsAmounts[_token], Comparators.gt);
    discountsAmounts[_token] = sortedAmounts;
    discounts[_token][discount_price] = discount_amount;
    updateCurrentDiscount(_token);
    
    emit SetDiscount(_token, discount_price, discount_amount);
}
```
