## Description
The `executeOrder()` function will check if an order is an `ftmSwap`. If so it will execute
differently in order to pay fees. However, the check for `ftmSwap` is incorrectly
implemented.

## Vulnerability Details
Looking at the `executeOrder()` function shown below:
```solidity
function executeOrder(
    uint256 id,
    uint256 amountTokenInGelatoFees,
    ExactInputParams memory dcaArgs,
    ExactInputParams memory ftmSwapArgs
) public onlyOwnerOrDedicatedMsgSender {
    // H-03 (onlyOwnerOrDedicatedMsgSender)
    //rest of the function
    if (
        ftmSwapArgs.path.length == 0 ||
        ftmSwapArgs.amountIn == 0 ||
        ftmSwapArgs.recipient == address(0)
    )
        // G-02
        revert ErrorInvalidExactInputParams();
    bool isFtmSwap = (ftmSwapArgs.path.length != 0 &&
        ftmSwapArgs.amountIn != 0) || ftmSwapArgs.recipient != address(0);
    if (isFtmSwap) {
        uint256 gelatoFees = 0;
        require(
            amountTokenInGelatoFees < ordersById[id].amountIn,
            "amountTokenInGelatoFees too high"
        );
        ordersById[id].amountIn -= amountTokenInGelatoFees;
        gelatoFees = ftmSwapArgs.amountIn;
        ftmSwapArgs.recipient = payable(address(this));
        SilverDcaApprover(ordersById[id].approver).transferGelatoFees(
            gelatoFees
        );
        TransferHelper.safeApprove(
            ordersById[id].tokenIn,
            address(swapRouter),
            gelatoFees
        );
        // L-06
        //ERC20(ordersById[id].tokenIn).approve(address(proxy),gelatoFees);
        swapRouter.exactInput(ftmSwapArgs);
    }
    _executeOrder(id, dcaArgs);
    if (isFtmSwap) {
        ordersById[id].amountIn = initialAmountIn;
        (uint256 fee, address feeToken) = _getFeeDetails();
        _transfer(fee, feeToken);
        emit GelatoFeesCheck(fee, feeToken);
    }
}
```
The highlighted part checks if the inputted swap is an `ftmSwap`. This logic is
implemented incorrectly.

In an example scenario: if the `ftmSwapArgs.path.length` is 0 and the
`ftmSwapArgs.amountIn` is 0, if the `ftmSwapArgs.reciepient` is not `address(0)`,
the `isFtmSwap` bool will be true. Meaning that even with incorrect inputs, function will
execute as if the bool is true.

## Impact
Function will calculate the `isFtmSwap` bool incorrectly.

## Recommendation
Change the logic as shown below:
```solidity
function executeOrder(uint256 id, uint256 amountTokenInGelatoFees,
ExactInputParams memory dcaArgs, ExactInputParams memory ftmSwapArgs) public
onlyOwnerOrDedicatedMsgSender {
  //rest of the function
  bool isFtmSwap = ftmSwapArgs.path.length != 0 && ftmSwapArgs.amountIn != 0
  && ftmSwapArgs.recipient != address(0);
  //rest of the function
}
```
