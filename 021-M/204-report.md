0xRajeev

high

# `commitToLiens` always reverts

## Summary

The function `commitToLiens()` always reverts at the call to `_returnCollateral()` which prevents borrowers from depositing collateral and requesting loans in the protocol.

## Vulnerability Detail

The collateral token with `collateralId` is already minted directly to the caller (i.e. borrower) in `commitToLiens()` at the call to `_transferAndDepositAsset()` function. That's because while executing `_transferAndDepositAsset` the NFT is transferred to `COLLATERAL_TOKEN` whose `onERC721Received` mints the token with `collateralId` to borrower (`from` address) and not the `operator_` (i.e. `AstariaRouter`) because `operator_ != from_`.

However, the call to `_returnCollateral()` in `commitToLiens()` incorrectly assumes that this has been minted to the operator and attempts to transfer it to the borrower which will revert because the `collateralId` is not owned by  `AstariaRouter` as it has already been transferred/minted to the borrower.

## Impact

The function `commitToLiens()` always reverts, preventing borrowers from depositing collateral and requesting loans in the protocol, thereby failing to bootstrap its core NFT lending functionality.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L244-L274
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L578-L587
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/CollateralToken.sol#L282-L284
4. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L589-L591

## Tool used

Manual Review

## Recommendation
Remove the call to `_returnCollateral()` in `commitToLiens()`.