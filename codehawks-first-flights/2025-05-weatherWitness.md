### [M-#] The `WeatherNft::fulfillMintRequest` function uses `_mint`, which omits recipient contract safety checks

# Root + Impact

## Description

The `WeatherNft::fulfillMintRequest` function mints an NFT based on weather conditions at a specified location.

It calls the low-level `_mint` function directly instead of the safer `_safeMint` variant. This approach is discouraged because `_mint` does not verify whether the recipient address can properly handle ERC721 tokens. If the recipient is a smart contract that does not implement the `IERC721Receiver` interface, the NFT may become permanently locked within that contract.

```solidity
function fulfillMintRequest(bytes32 requestId) external {
        bytes memory response = s_funcReqIdToMintFunctionReqResponse[requestId].response;
        bytes memory err = s_funcReqIdToMintFunctionReqResponse[requestId].err;

        require(response.length > 0 || err.length > 0, WeatherNft__Unauthorized());

        if (response.length == 0 || err.length > 0) {
            return;
        }

        UserMintRequest memory _userMintRequest = s_funcReqIdToUserMintReq[
            requestId
        ];
        uint8 weather = abi.decode(response, (uint8));
        uint256 tokenId = s_tokenCounter;
        s_tokenCounter++;

        emit WeatherNFTMinted(
            requestId,
            msg.sender,
            Weather(weather)
        );
@>      _mint(msg.sender, tokenId);
        s_tokenIdToWeather[tokenId] = Weather(weather);



        // rest of the function code


    }
```

## Risk

**Likelihood**:

- The problem arises when the token is minted by a contract that does not handle its transfer correctly.

**Impact**:

- Potential for NFTs to be irretrievably locked if minted to a contract that does not properly handle ERC721 transfers.

## Recommended Mitigation

Use the `_safeMint` function instead of `_mint` to ensure that the recipient address can safely handle ERC721 tokens.

```diff
    function fulfillMintRequest(bytes32 requestId) external {
        bytes memory response = s_funcReqIdToMintFunctionReqResponse[requestId].response;
        bytes memory err = s_funcReqIdToMintFunctionReqResponse[requestId].err;

        require(response.length > 0 || err.length > 0, WeatherNft__Unauthorized());

        if (response.length == 0 || err.length > 0) {
            return;
        }

        UserMintRequest memory _userMintRequest = s_funcReqIdToUserMintReq[
            requestId
        ];
        uint8 weather = abi.decode(response, (uint8));
        uint256 tokenId = s_tokenCounter;
        s_tokenCounter++;

        emit WeatherNFTMinted(
            requestId,
            msg.sender,
            Weather(weather)
        );
-       _mint(msg.sender, tokenId);
+       _safeMint(msg.sender, tokenId);
        s_tokenIdToWeather[tokenId] = Weather(weather);


        // rest of the function code

    }

```

The `_safeMint` function performs an additional check by calling `onERC721Received` on the recipient contract to confirm that it can handle ERC721 tokens.

### [H-#] The `WeatherNft` contract includes a payable function but does not implement a mechanism to withdraw the received Ether

# Root + Impact

## Description

- The `WeatherNft::requestMintWeatherNFT` function has `payable` modifier and requires sending Ether to mint an NFT. The required payment increases with the number of tokens already minted.
- The `WeatherNft` contract does not include any function for withdrawing Ether, so all funds paid for minted tokens remain locked in the contract.

```solidity
    function requestMintWeatherNFT(
        string memory _pincode,
        string memory _isoCode,
        bool _registerKeeper,
        uint256 _heartbeat,
        uint256 _initLinkDeposit
@>  ) external payable returns (bytes32 _reqId) {
        // rest of the function code
    }
```

## Risk

**Likelihood** and **Impact**:

- All Ether paid by users for minted tokens will remain permanently locked in the contract.

## Recommended Mitigation

Add a withdrawal function that can only be executed by the contract owner. For example:

```diff
+   function withdraw() external OnlyOwner {
+       uint256 amount = address(this).balance;
+       require(amount > 0, "No funds to withdraw");
+       (bool success, ) = msg.sender.call{value: amount}("");
+       require(success, "Failure! Ether not sent!");
+   }
```
