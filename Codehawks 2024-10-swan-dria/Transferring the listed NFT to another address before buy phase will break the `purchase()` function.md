## Summary

It is possible for a malicious `seller` to `list` an NFT and transfer it to any address that is not the seller address. This will cause the `purchase` function to always revert, breaking the contract logic, griefing all honest `seller`s and the `buyerAgent`.

## Vulnerability Details

Any seller can `list` an NFT with the `list` function.

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

As observed in this function, when it is called, it deploys a new NFT contract.

```Solidity
// SPDX-License-Identifier: Apache-2.0
pragma solidity ^0.8.20;

import {ERC721} from "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

/// @notice Factory contract to deploy SwanAsset tokens.
/// @dev This saves from contract space for Swan.
contract SwanAssetFactory {
    /// @notice Deploys a new SwanAsset token.
    function deploy(string memory _name, string memory _symbol, bytes memory _description, address _owner)
        external
        returns (SwanAsset)
    {
        return new SwanAsset(_name, _symbol, _description, _owner, msg.sender);
    }
}

/// @notice SwanAsset is an ERC721 token with a single token supply.
contract SwanAsset is ERC721, Ownable {
    /// @notice Creation time of the token
    uint256 public createdAt;
    /// @notice Description of the token
    bytes public description;

    /// @notice Constructor sets properties of the token.
    constructor(
        string memory _name,
        string memory _symbol,
        bytes memory _description,
        address _owner,
        address _operator
    ) ERC721(_name, _symbol) Ownable(_owner) {
        description = _description;
        createdAt = block.timestamp;

        // owner is minted the token immediately
        ERC721._mint(_owner, 1);

        // Swan (operator) is approved to by the owner immediately.
        ERC721._setApprovalForAll(_owner, _operator, true);
    }
}
```

This newly deployed contract will `mint` an NFT for the seller and set the `Swan.sol` contract as the `operator`. Take notice that the `_update` function of `ERC721.sol` is not overriden, meaning there are no changes done to the transfer logic of this NFT.

Assume that the price of this NFT is set  low so this NFT is selected to be bought by the `buyerAgent`  as it is a very profitable purchase. Malicious `seller` can also set the price so low (e.g. 0) that he does not have to pay any royalties making this attack cost nothing but gas. Here, malicious `seller` has succesfully blocked at minimum one user from selling their NFT to the `buyerAgent`. 

While it is still the sell phase, malicious `seller` will transfer the NFT to any address. 

When it reaches the buy phase, `buyerAgent` will attempt to buy  NFTs with the `purchase` function in `BuyerAgent.sol`.

```Solidity
    function purchase() external onlyAuthorized {
        // check that we are in the Buy phase, and return round
        (uint256 round,) = _checkRoundPhase(Phase.Buy);

        // check if the task is already processed
        uint256 taskId = oraclePurchaseRequests[round];
        if (isOracleRequestProcessed[taskId]) {
            revert TaskAlreadyProcessed();
        }

        // read oracle result using the latest task id for this round
        bytes memory output = oracleResult(taskId);
        address[] memory assets = abi.decode(output, (address[]));

        // we purchase each asset returned
        for (uint256 i = 0; i < assets.length; i++) {
            address asset = assets[i];

            // must not exceed the roundly buy-limit
            uint256 price = swan.getListingPrice(asset);
            spendings[round] += price;
            if (spendings[round] > amountPerRound) {
                revert BuyLimitExceeded(spendings[round], amountPerRound);
            }

            // add to inventory
            inventory[round].push(asset);

            // make the actual purchase
            swan.purchase(asset); // WILL REVERT HERE
        }

        // update taskId as completed
        isOracleRequestProcessed[taskId] = true;
    }
```

This function will reach the malicious seller's asset in the for loop and it will call the `purchase` function in `Swan.sol`.

```Solidity
    function purchase(address _asset) external {
        AssetListing storage listing = listings[_asset];

        // asset must be listed to be purchased
        if (listing.status != AssetStatus.Listed) {
            revert InvalidStatus(listing.status, AssetStatus.Listed);
        }

        // can only the buyer can purchase the asset
        if (listing.buyer != msg.sender) {
            revert Unauthorized(msg.sender);
        }

        // update asset status to be sold
        listing.status = AssetStatus.Sold;

        // transfer asset from seller to Swan, and then from Swan to buyer
        // this ensure that only approval to Swan is enough for the sellers
        SwanAsset(_asset).transferFrom(listing.seller, address(this), 1); // WILL REVERT HERE
        SwanAsset(_asset).transferFrom(address(this), listing.buyer, 1);

        // transfer money
        token.transferFrom(listing.buyer, address(this), listing.price);
        token.transfer(listing.seller, listing.price);

        emit AssetSold(listing.seller, msg.sender, _asset, listing.price);
    }
```

This function will attempt to transfer the NFT from the `seller` address to the `Swan.sol` contract with the `transferFrom` function.

```Solidity
    function transferFrom(address from, address to, uint256 tokenId) public virtual {
        if (to == address(0)) {
            revert ERC721InvalidReceiver(address(0));
        }
        // Setting an "auth" arguments enables the `_isAuthorized` check which verifies that the token exists
        // (from != 0). Therefore, it is not needed to verify that the return value is not 0 here.
        address previousOwner = _update(to, tokenId, _msgSender());
        if (previousOwner != from) {
            revert ERC721IncorrectOwner(from, tokenId, previousOwner); //WILL REVERT HERE
        }
    }
```

As the `previousOwner` is no longer matching `from(seller)` address, this call will revert. Making it impossible for the `buyerAgent` to purchase this NFT or any of the NFTs that it wants to purchase as the main `purchase` call will revert. Malicious `seller` has now blocked all honest `seller`s from selling their NFT with no cost other than gas and has succesfully griefed the `buyerAgent`.

## Impact

Impact: High, malicious actors will very easily grief honest `seller`s and the `buyerAgent`, breaking contract logic. `buyerAgent` will not be able to purchase any NFT due to this vulnerability.\
Likelihood: Medium, this attack is very easy to do and costs nothing but gas for the attacker. However, even if the attacker sets the price to 0 they can not guarantee their NFT will be chosen to be purchased.

## Tools Used

Manual review, Hardhat

## Recommendations

During NFT listing with the `list` function, `transfer` the NFT to the `Swan.sol` contract. Send these NFT's back to the `seller` upon unsuccesful sell phase by implementing a new function. Implement the same logic in the `relist` function aswell and update the `purchase` logic and remove the first `transferFrom` line. An example is shown below.

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
        SwanAsset(asset).transferFrom(listing.seller, address(this), 1); //Added line
        //rest of the function
```
