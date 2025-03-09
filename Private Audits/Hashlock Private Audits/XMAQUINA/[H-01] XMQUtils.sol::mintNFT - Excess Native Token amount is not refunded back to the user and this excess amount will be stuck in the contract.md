## Description
In order for a user to mint the NFT to join the presale, they must call the `mintNFT` function in the `XMQUtils.sol` contract. This contract transfers the minting cost to the treasury address but does not consider scenarios where users might send more native tokens than this cost. The excess amount that is sent to the contract will forever be stuck.

## Vulnerability Details
The `mintNFT` function shown below mints a whitelist NFT to the user calling it.

```solidity
function mintNFT() payable external {
    if (address(whitelistNFT) == address(0)) revert Errors.WhitelistNFTZeroAddr();
    if (!isWhitelisted[msg.sender]) revert Errors.NotWhitelisted();
    if (hasMintedNFT[msg.sender]) revert Errors.AlreadyMinted();

    if (paymentToken == NATIVE_TOKEN_ADDRESS) {
        if (msg.value < mintingCost) revert Errors.InsufficientFunds();
        // Transfer the native token (ETH) to the contract's treasury
        (bool sent, ) = treasuryAddress.call{value: mintingCost}("");
        if (!sent) revert Errors.EtherTransferFailed();
    } else {
        IERC20(paymentToken).transferFrom(msg.sender, treasuryAddress, mintingCost);
    }

    hasMintedNFT[msg.sender] = true;
    XMQWhitelistNFT(whitelistNFT).mint(msg.sender);
    emit NFTMinted(msg.sender);
}
```

As seen from the highlighted parts, if the `paymentToken` is the Native Token, the function checks if the `msg.value` amount is equal or greater than the `mintingCost`, function will send the `mintingCost` amount to the `treasuryAddress`. In a scenario where a user has sent a greater `msg.value` than the `mintingCost`, excess native tokens will not be refunded to the user. Since the contract lacks any function to withdraw native tokens in the contract, these excess tokens will forever be stuck in the contract. 

## Impact
Users that end up paying more than the `mintingCost` will not be refunded the excess amount. This excess amount will forever be stuck in the contract.

## Recommendation
Implement refund of excess amounts in the function. An example is shown below.

```solidity
function mintNFT() payable external {
    if (address(whitelistNFT) == address(0)) revert Errors.WhitelistNFTZeroAddr();
    if (!isWhitelisted[msg.sender]) revert Errors.NotWhitelisted();
    if (hasMintedNFT[msg.sender]) revert Errors.AlreadyMinted();

    if (paymentToken == NATIVE_TOKEN_ADDRESS) {
        if (msg.value < mintingCost) revert Errors.InsufficientFunds();
        
        // Transfer the native token (ETH) to the contract's treasury
        (bool sent, ) = treasuryAddress.call{value: mintingCost}("");
        if (!sent) revert Errors.EtherTransferFailed();

        // Refund excess `msg.value`
        uint256 excess = msg.value - mintingCost;
        if (excess > 0) {
            (bool refundSent, ) = msg.sender.call{value: excess}("");
            if (!refundSent) revert Errors.RefundFailed();
        }
    } else {
        IERC20(paymentToken).transferFrom(msg.sender, treasuryAddress, mintingCost);
    }

    hasMintedNFT[msg.sender] = true;
    XMQWhitelistNFT(whitelistNFT).mint(msg.sender);
    emit NFTMinted(msg.sender);
}
```

Alternatively, make this function `nonReentrant` and implement a new function to withdraw the native tokens in the contract.

