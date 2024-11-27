## Description
In the `executeOrder()` function, execution of swaps will happen differently if the
inputted order is an `ftmSwap` or not. However, due to incorrect logic, any order that is
not an `ftmSwap` will revert the function.

## Vulnerability Details
In the `executeOrder()` function shown below:
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
It is observed that, if the inputted order is an `ftmSwap`, this function will execute
differently and require a payment of fees. However, due to the highlighted part in the
code snippet, any swap that is not `ftmSwap` will cause the function to revert.

## Impact
This vulnerability causes any calls to this function when it is not ftmSwap to fail.

## Recommendation
Remove the highlighted revert logic.
