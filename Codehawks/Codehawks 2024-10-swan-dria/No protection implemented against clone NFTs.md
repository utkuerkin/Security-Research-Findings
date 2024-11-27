## <a id='H-01'></a>H-01. No protection implemented against listing clone NFTs

_Submitted by [foxb868](https://profiles.cyfrin.io/u/undefined), [ljj](https://profiles.cyfrin.io/u/undefined), [ChainDefenders](https://codehawks.cyfrin.io/team/cm2bxupf00003grinaqv78qfm), [n3smaro](https://profiles.cyfrin.io/u/undefined). Selected submission by: [ljj](https://profiles.cyfrin.io/u/undefined)._      
            


## Summary

Any malicious seller can copy the name, symbol and decription of any previously listed asset and list it at a 1 wei lower price. In the case of this NFT being selected to be purchased by the system, this malicious seller will guarantee that their asset with lower price would be selected. This will lead to user's copying previously listed NFT's and listing it at a lower price, in a way creating a price race to the bottom.

## Vulnerability Details

Any seller can list an NFT with the `list` function.

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

As observed, this function take the `_name`, `_symbol`, `_desc` and `_price` parameters. Function then deploys a new NFT with these parameters  and mints 1 NFT to the `msg.sender`.

```Solidity
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

As observed, there are no checks for "clone" inputs. Meaning that any malicious seller can copy the parameters of any previously listed NFT and list it at a 1 wei lower price. In case of an NFT with these parameters being selected, the malicious user would guarantee their NFT at lower price would be selected, putting no work in creating an original NFT and simply copying previously deployed NFTs. This will create a price race to the bottom among users where users would list the NFT with same parameters, each one putting it at 1 wei lower price, breaking the protocols intended use. The AI agents seeing all or most of the NFT's listed having the same properties would choose to purchase the NFT with these properties at the lowest price.

## Impact

Impact: High, this vulnerability will break the intended use of the protocol. It will create a price race to the bottom where users list the NFT with same name, symbol and description, each user listing it at 1 wei lower price to ensure their NFT would be chosen by the system.\
Likelihood: Low, There are no guarantees that this NFT would be chosen by the system but noticing copies of the same NFTs can manipulate the LLM into thinking this is a good purchase.

## Tools Used

Manual review

## Recommendations

Implement a for loop in the `list` function that will check the NFT that is being listed against the already listed NFTs. An example for loop is shown below. Keep in mind that this implementation might cost a lot of gas if there are too many listed NFTs.

```Solidity
       address[] memory assets = assetsPerBuyerRound[buyer][round];
       for (uint256 i = 0; i < assets.length; i++) {
            IERC721 asset = IERC721(assets[i]);
            
            // Retrieve the name and symbol from the asset contract
            string memory assetName = asset.name();
            string memory assetSymbol = asset.symbol();
            
            // Check if both name and symbol match
            if (keccak256(bytes(assetName)) == keccak256(bytes(targetName)) &&
                keccak256(bytes(assetSymbol)) == keccak256(bytes(targetSymbol))) {
                revert("Asset with matching name and symbol already exists in this round");
            }
        }
```
