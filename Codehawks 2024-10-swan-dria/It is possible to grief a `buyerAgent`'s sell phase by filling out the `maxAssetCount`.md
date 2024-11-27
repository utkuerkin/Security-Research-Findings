## Summary

A malicious `seller` can grief a `buyerAgent` by filling out the `maxAssetsCount`. Such attack is supposed to be discouraged by the royalties that the seller has to pay, however creating a listing with a low `_price` input (e.g. 0 or 1) will cause the royalties to be 0 meaning that it will cost nothing but gas fees for the attacker.

## Vulnerability Details

Any `seller` can call the `list` function to deploy, mint and list a Swan Asset NFT

```Solidity
    function list(string calldata _name, string calldata _symbol, bytes calldata _desc, uint256 _price, address _buyer)
        external
    {
        BuyerAgent buyer = BuyerAgent(_buyer);
        (uint256 round, BuyerAgent.Phase phase,) = buyer.getRoundPhase();

        // buyer must be in the sell phase
        if (phase != BuyerAgent.Phase.Sell) {
            revert BuyerAgent.InvalidPhase(phase, BuyerAgent.Phase.Sell);
        }
        // asset count must not exceed `maxAssetCount`
        if (getCurrentMarketParameters().maxAssetCount == assetsPerBuyerRound[_buyer][round].length) {
            revert AssetLimitExceeded(getCurrentMarketParameters().maxAssetCount);
        }

        // all is well, create the asset & its listing
        address asset = address(swanAssetFactory.deploy(_name, _symbol, _desc, msg.sender));
        listings[asset] = AssetListing({
            createdAt: block.timestamp,
            royaltyFee: buyer.royaltyFee(),
            price: _price,
            seller: msg.sender,
            status: AssetStatus.Listed,
            buyer: _buyer,
            round: round
        });

        // add this to list of listings for the buyer for this round
        assetsPerBuyerRound[_buyer][round].push(asset);

        // transfer royalties
        transferRoyalties(listings[asset]);

        emit AssetListed(msg.sender, asset, _price);
    }
```

As observed in this function, this function will deploy, mint and list the NFT as long as the `buyer` is in sell phase and the sell round has not reached maximum amount of listings(`maxAssetCount`). \
This function will then call the `transferRoyalties` function.

```Solidity
    function transferRoyalties(AssetListing storage asset) internal {
        // calculate fees
        uint256 buyerFee = (asset.price * asset.royaltyFee) / 100;
        uint256 driaFee = (buyerFee * getCurrentMarketParameters().platformFee) / 100;

        // first, Swan receives the entire fee from seller
        // this allows only one approval from the seller's side
        token.transferFrom(asset.seller, address(this), buyerFee);

        // send the buyer's portion to them
        token.transfer(asset.buyer, buyerFee - driaFee);

        // then it sends the remaining to Swan owner
        token.transfer(owner(), driaFee);
    }
```

As seen from these functions, `asset.price` has no lower limit meaning that it can be set to a low number such as 0 or 1. If a malicious seller sets the price to 0 or 1, even if the `royaltyFee` and `platformFee` are set to 99, due to how solidity handles divisions, `buyerFee` and `driaFee` will be set to 0. \
Considering all of the above, a malicious `seller` can spam create listings via calling the `list` function with `_price` set to 0 or 1 to fill out the `maxAssetCount` of the `buyerAgent` for that sell period. Due to the royalty math, this malicious `seller` would also pay no fees to the protocol or the `buyerAgent` and only pay the gas fee for the transactions. This will lead to the `maxAssetCount` reaching the maximum amount preventing any honest `seller`s from listing their NFT. Effectively griefing that `buyerAgent` permanently, causing no NFTs to be able to be listed.

## Proof of Concept

Adjust the `Swan.test.ts` to mimic listings with `_price` set to 0. \
1\) Add the following line in the test file

```JavaScript
 const PRICE4 = parseEther("0.00");
```

2\) Adjust the following test as shown below

```JavaScript
    it("should list 5 assets for the first round", async function () {
      await listAssets(
        swan,
        buyerAgent,
        [
          [seller, PRICE4],
          [seller, PRICE4],
          [seller, PRICE4],
          [sellerToRelist, PRICE4],
          [sellerToRelist, PRICE4],
        ],
        NAME,
        SYMBOL,
        DESC,
        0n
      );

      [assetToBuy, assetToRelist, assetToFail, ,] = await swan.getListedAssets(
        await buyerAgent.getAddress(),
        currRound
      );

      expect(await token.balanceOf(seller)).to.be.equal(FEE_AMOUNT1 + FEE_AMOUNT2 + FEE_AMOUNT3 + FEE_AMOUNT1 + FEE_AMOUNT2);
      expect(await token.balanceOf(sellerToRelist)).to.be.equal(FEE_AMOUNT2 + FEE_AMOUNT1);
    });
```

This test proves that it is possible to create listings with `_price` as 0, filling out the `maxAssetCount`.

## Impact

Impact: High, the buyerAgent will be permanently griefed and will not be able to purchase any honest NFTs\
Likelihood: High, attack costs no fees to the seller and it is easy to perform.

## Tools Used

Manual review, hardhat

## Recommendations

Implement a minimum fee for all listings to discourage these kind of griefing attacks. An example is shown below.

```Solidity
    function transferRoyalties(AssetListing storage asset) internal {
        // calculate fees
        uint256 buyerFee = (asset.price * asset.royaltyFee) / 100;
        if (buyerFee == 0) {
            buyerFee = 10;
        }
       // rest of the function
```

Alternatively, implement a minimum selling price. An example is shown below.

```Solidity
function list(string calldata _name, string calldata _symbol, bytes calldata _desc, uint256 _price, address _buyer)
        external
    {
        if(_price < 100){
           revert; 
        }
       // rest of the function
      
function relist(address _asset, address _buyer, uint256 _price) external {
        if(_price < 100){
           revert; 
        }
       // rest of the function
```
