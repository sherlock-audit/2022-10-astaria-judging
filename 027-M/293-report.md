Rohan16

medium

# Use safeTransferFrom() instead of transferFrom() for outgoing erc721 transfer

## Summary
It is recommended to use safeTransferFrom() instead of transferFrom() when transferring ERC721s out of the vault.
## Vulnerability Detail

1. The transferFrom() method is used instead of safeTransferFrom(), which I assume is a gas-saving measure. I however argue that this isn’t recommended because:
- [OpenZeppelin’s documentation](https://docs.openzeppelin.com/contracts/4.x/api/token/erc721#IERC721-transferFrom-address-address-uint256-) discourages the use of transferFrom(); use safeTransferFrom() whenever possible
- The recipient could have logic in the onERC721Received() function, which is only triggered in the safeTransferFrom() function and not in transferFrom(). A notable example of such contracts is the Sudoswap pair:

## Impact
While unlikely because the recipient is the function caller, there is the potential loss of NFTs should the recipient is unable to handle the sent ERC721s.
[IERC721(underlyingAsset).transferFrom(address(this), releaseTo, assetId);](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/CollateralToken.sol#L223)
[nft.transferFrom(address(this), address(receiver), tokenId);](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/CollateralToken.sol#L176)
## Code Snippet

```solidity
nft.transferFrom(address(this), address(receiver), tokenId);
 IERC721(underlyingAsset).transferFrom(address(this), releaseTo, assetId);
```
## Tool used

Manual Review
## Recommendation
Use safeTransferFrom() when sending out the NFT from the vault.
```solidity 
- IERC721(_erc721Address).transferFrom(address(this), msg.sender, _id);
+ IERC721(_erc721Address).safeTransferFrom(address(this), msg.sender, _id);
```
Note that the vault would have to inherit the IERC721Receiver contract if the change is applied to Transfer.sol as well.

