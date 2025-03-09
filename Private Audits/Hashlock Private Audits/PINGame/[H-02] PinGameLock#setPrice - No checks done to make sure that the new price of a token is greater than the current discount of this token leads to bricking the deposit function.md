## Description
The `setPrice` function allows the owner to set the price of a token. However, similar to **[H-01]**, this function lacks checks to ensure that the price remains higher than the discount. When the price of a token is set to an amount lower than the current discount, it will brick the `deposit` function, as the `actualPrice` calculation will always revert due to an underflow. This vulnerability also breaks the `prices` view function.

## Vulnerability Details
The `setPrice` function allows the owner to set the price of a token.

```solidity
function setPrice(address _token,uint256 _price) onlyOwner external {
    require(supportedTokens[_token],"token not exist");

    uint256 oldPrice = originPrices[_token];
    originPrices[_token] = _price;
    emit SetPrice(_token, oldPrice, _price);
}
```

As observed, the only check done in this function is to ensure that the token inputted is a supported token. This means that the price of a token can be set to an amount that is lower than the discount.

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

If this token's discount is higher than the price, this calculation will cause a revert due to an underflow. As there are no ways of removing discounts from the discount array, the only fix to this problem is updating the price of the token to be higher than the discount, but this fix is a workaround that breaks the integrity of the contract. This vulnerability also breaks the `prices` view function.

## Proof of Concept
Set up the test suite as shown in **[H-01]**.

Implement the following test and run it to observe the vulnerability:

```solidity
function testUnderflowPriceUpdate() public {
    vm.startPrank(owner);
    lock.setDiscount(address(token), 90 ether, 2);
    vm.stopPrank();

    vm.startPrank(user);
    // Try to deposit, it should work
    lock.deposit(address(token), 100 ether, "TEST");
    vm.stopPrank();

    vm.startPrank(owner);
    // Setting price lower than existing discount
    lock.setPrice(address(token), 50 ether);
    vm.stopPrank();

    vm.startPrank(user2);
    // Now try to deposit - this should revert due to underflow in price calculation
    vm.expectRevert(stdError.arithmeticError);
    lock.deposit(address(token), 100 ether, "TEST");
    vm.stopPrank();
}
```

## Impact
The `deposit` function, which is integral to this contract’s functionality, will be bricked permanently and will not work.

## Recommendation
Implement checks in the function to ensure that the new price of the token cannot be lower than the existing discount.
