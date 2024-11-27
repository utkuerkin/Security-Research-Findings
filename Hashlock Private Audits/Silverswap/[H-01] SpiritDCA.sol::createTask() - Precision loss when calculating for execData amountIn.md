## Description
In the `createTask()` function, `execData` takes necessary swap data and saves it in
strings. This data is then used in the `AutomateTaskCreator.sol::_createTask()`
function. However due to division before multiplication, the saved string value of
`amountIn` will be different from the intended result.

## Vulnerability Details
Spiritswap takes a 1% fee for orders, this is why in the `createTask()` function, 99% of
`amountIn` is calculated and stored in `execData` to be used in the swap as shown in the
code snippet below.
```solidity
function createTask(uint256 id) private {
  require(ordersById[id].taskId == bytes32(""), 'Task already created.');
  bytes memory execData = abi.encode(
    ...
    Strings.toString((ordersById[id].amountIn / 100) * 99),
    ...
}
```
Here, during the calculation of 99% of `amountIn` value, division is done before
multiplication which results in precision loss, this is because the division operation in
Solidity performs integer division, discarding any remainder.

## Proof of Concept
To see the loss of precision compare the results of these two functions:
```solidity
uint256 amountIn = 987987987;

function divisionBeforeMultiplication() public view returns (uint256){
    return (amountIn / 100) * 99;
}

function divisionAfterMultiplication() public view returns (uint256){
    return (amountIn * 99) / 100;
}
```
While the first function gives the result of `978108021`, second function will give
`978108107` which is closer to the actual 99% of `978978978` which is `978108107.13`.

## Impact
Precision loss causes incorrect calculation of `amountIn` that is sent to the swap. Users
will lose more tokens than they should based on the 1% fee that Spiritswap takes.

## Recommendation
Do the division after the multiplication as shown below.
```solidity
function createTask(uint256 id) private {
  require(ordersById[id].taskId == bytes32(""), 'Task already created.');
  bytes memory execData = abi.encode(
    ...
    Strings.toString((ordersById[id].amountIn * 99) * 100),
    ...
}
```


