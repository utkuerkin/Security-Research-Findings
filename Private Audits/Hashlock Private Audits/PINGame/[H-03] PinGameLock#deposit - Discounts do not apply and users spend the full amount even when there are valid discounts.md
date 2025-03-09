## Description
The `deposit` function allows users to deposit tokens into the contract. This function has a discount system implemented, but this discount does not change the amount that users have to spend. Even with valid discounts, users will spend the full amount, and their deposited balance will update with this amount.

## Vulnerability Details
The `deposit` function allows users to deposit tokens into the contract:

```solidity
function deposit(address _token, uint256 amount, string memory refererCode) external {
    require(supportedTokens[_token], "token not supported");
    require(amount > 0, "amount should be greater than 0");
    require(originPrices[_token] > 0, "price should be greater than 0");
    uint256 cd = getCurrentDiscount(_token);
    uint256 actualPrice = originPrices[_token] - cd;

    require(amount >= actualPrice, "should be greater than price");
    require(userBought[msg.sender] == false, "already bought");

    // Transfer the full amount from the user
    SafeERC20.safeTransferFrom(IERC20(_token), msg.sender, address(this), amount);

    userBalance[msg.sender][_token] += amount;
    userBought[msg.sender] = true;

    if (cd > 0 && amount < originPrices[_token]) { // user used the discount
        discounts[_token][cd] -= 1;
        updateCurrentDiscount(_token);
    }

    emit Deposit(msg.sender, _token, amount, refererCode);
}
```

As observed, there are calculations done to apply a discount and calculate the `actualPrice` of the token. However, even with this calculation, the full `amount` entered by the user is transferred from them, meaning the discounts are not applied correctly. Users still spend the full amount they input, and their deposited balance increases by this amount. The only way for users to make use of the discount is to know the price and the current discount and input the correct amount manually.

## Proof of Concept
Set up the test suite as shown in **[H-01]**.

Implement the following test and run it to observe the vulnerability:

```solidity
function testDiscountDoesNotApply() public {
    vm.startPrank(owner);
    // Setting two discounts
    lock.setDiscount(address(token), 90 ether, 1);
    lock.setDiscount(address(token), 80 ether, 1);
    vm.stopPrank();

    uint256 user1BalanceBefore = token.balanceOf(user);
    // Deposit with user1
    vm.startPrank(user);
    lock.deposit(address(token), 100 ether, "TEST");
    vm.stopPrank();
    uint256 user1BalanceAfter = token.balanceOf(user);

    uint256 user2BalanceBefore = token.balanceOf(user2);
    // Deposit with user2
    vm.startPrank(user2);
    lock.deposit(address(token), 99 ether, "TEST");
    vm.stopPrank();
    uint256 user2BalanceAfter = token.balanceOf(user2);

    uint256 user1Deposited = lock.userBalance(user, address(token));
    uint256 user2Deposited = lock.userBalance(user2, address(token));
    
    // Assertions to make sure both users deposited the correct amount
    assertEq(user1Deposited, 100 ether);
    assertEq(user2Deposited, 99 ether);

    // Assertions prove both users spent the full amount
    assertEq(user1BalanceBefore - user1BalanceAfter, 100 ether);
    assertEq(user2BalanceBefore - user2BalanceAfter, 99 ether);
}
```

## Impact
The discount functionality is not implemented correctly. Even with valid discounts present, users still pay the full amount.

## Recommendation
Change the function to take discounts into consideration and correctly calculate the amount of tokens users must send to the contract.
Alternatively, do not take user input for the amount to send and let the function calculate this amount and transfer it from the user.

