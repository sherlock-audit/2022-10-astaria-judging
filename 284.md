HonorLt

medium

# Re-entrancy

## Summary
The protocol does not have safety checks against re-entrancy attacks.

## Vulnerability Detail
The protocol is quite big and complex and there are many outside interactions with users or other contracts containing callbacks. For example, users can ```flashAction``` their NFTs if they return them in the same transaction. I believe this makes it possible to re-use the same collateral twice. For instance, when the control is given back to the user:
```solidity
    require(
      receiver.onFlashAction(IFlashAction.Underlying(addr, tokenId), data) ==
        keccak256("FlashAction.onFlashAction"),
      "flashAction: callback failed"
    );
```
a malicious actor can ```commitToLien``` again (combining with issues that the ```ecrecover``` might be bypassed and a strategist nonce is not invalidated). Because the user already owns the collateral token, the contract will skip the minting and return:
```solidity
  function onERC721Received(
    address operator_,
    address from_,
    uint256 tokenId_,
    bytes calldata data_
  ) external override returns (bytes4) {
    uint256 collateralId = msg.sender.computeId(tokenId_);

    (address underlyingAsset, ) = getUnderlying(collateralId);
    if (underlyingAsset == address(0)) {
      ...
    }
    return IERC721Receiver.onERC721Received.selector;
  }
```

## Impact
I believe it might be possible to reuse the same collateral twice. I was short on time to PoC this and there might be more places where re-entrancy can be exploited. The codebase is just quite complicated to make a sufficient mental model about the intended behavior.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/CollateralToken.sol#L149-L197

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/CollateralToken.sol#L259-L296

## Tool used

Manual Review

## Recommendation
Add re-entrancy guard to critical functions that contain external interactions.
