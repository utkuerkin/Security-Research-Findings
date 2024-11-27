## Description
Any malicious attacker can call `executeOrder()` function with the parameters they
want to enter.

## Vulnerability Details
Consider the scenario where an attacker will call `executeOrder()` function with the
id of any other user. Attacker here can enter their desired `ftmSwapArgs` into this
function.
```solidity
if (isFtmSwap)
  {
  uint256 gelatoFees = 0;
  if (!isSimpleDataEmpty(ftmSwapArgs.simpleData)) {
    gelatoFees = ftmSwapArgs.simpleData.fromAmount;
    ordersById[id].amountIn -= gelatoFees;
  } else if (!isSellDataEmpty(ftmSwapArgs.sellData)) {
    gelatoFees = ftmSwapArgs.sellData.fromAmount;
    ordersById[id].amountIn -= gelatoFees;
  } else if (!isMegaSwapSellDataEmpty(ftmSwapArgs.megaSwapSellData)) {
    gelatoFees = ftmSwapArgs.megaSwapSellData.fromAmount;
    ordersById[id].amountIn -= gelatoFees;
  }
  SpiritDcaApprover(ordersById[id].approver).transferGelatoFees(gelatoFees);
  ERC20(ordersById[id].tokenIn).approve(address(proxy), gelatoFees);
```
In the code snippet above, notice that the attacker can set the
`ftmSwapArgs.simpleData.fromAmount` to any amount that is <= `amountIn` to
avoid underflow. Here, assume that the attacker set the `gelatoFees` to the max
amount they can.

This function will then call the `DcaApprover.sol::transferGelatoFees()` function
with the amount that the attacker has set.
```solidity
function transferGelatoFees(uint256 feesAmount) public {
    require(msg.sender == dca, "Only DCA can execute order.");
    Order memory order = ISpiritDCA(dca).ordersById(id);
    require(
        block.timestamp - order.lastExecution >= order.period,
        "Period not elapsed."
    );
    require(!order.stopped, "Order is stopped.");
    TransferHelper.safeTransferFrom(tokenIn, user, dca, feesAmount);
}
```
Notice that the `transferGelatoFees()` function shown above will transfer the
amount set by the attacker, from the user into the `SpiritDCA.sol` contract.

Lack of access control here causes similar, dangerous vulnerabilities when `dcaArgs` set
by the attacker is put into `_executeOrder()` function, or when the `executeOrder()`
function calls `proxy.simpleSwap()`.

## Impact
An attacker will cause a user to lose funds.

## Recommendation
Implement access control into the `executeOrder()` function so only the automation
contract or the owner of the order can call it. For example: the
`AutomateReady.sol::onlyDedicatedMsgSender()` modier. More detailed
approach on this in [Gelato docs](https://docs.gelato.network/web3-services/web3-functions/security-considerations).

Additionally, implement restrictions for `ftmSwapArgs` and `dcaArgs`.

