yixxas

medium

# `createAuction()` does not check if an auction is already existing

## Summary
`createAuction()` does not check if an auction is already existing and simply overwrites it if it does exist. This is extremely problematic as it essentially erases all bids made by users as well as erases all storage of the previous auction and replace them with the initialized value.

## Vulnerability Detail
An auction can be created in 2 ways, one with `auctionVault()` that calls `createAuction()` that does the relevant check and another using the auction house directly with `createAuction()`. I believe `auctionVault()` is intended to be the correct way to create an auction. But we should not discount the possibility of `createAuction()` being called directly by mistake, as it can lead to severe unintended consequences.

## Impact
An auction that is created with a `tokenId` over an already existing one can lead to loss of assets of users as the "new" auction storage is replaced with initial values.

## Code Snippet
https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/CollateralToken.sol#L310-L325
https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L67-L85

## Tool used

Manual Review

## Recommendation
The check if an auction is already existing should be done in `createAuction()` instead of `auctionVault()`. This way, calling either function still ensures that this important check is done.
