0xRajeev

medium

# Auctions run for less time than intended

## Summary

Auctions run for less time than intended causing them to be not economically efficient for the lenders, thus causing a suboptimal credit of liquidation funds to them.

## Vulnerability Detail

From the documentation/comments, we can infer that `firstBidTime` is supposed to be `// The time of the first bid` and duration is supposed to be `// The length of time to run the auction for, after the first bid was made.`

However, when an auction is created in `createAuction()`, the auction's `firstBidTime` is incorrectly initialized as `block.timestamp.safeCastTo64()` instead of `0`. This is premature initialization because the auction was only created here and no bid has been made yet via `createBid()`. The code in `createBid()` which check for `firstBidTime == 0 `is more evidence that the initialization in `createAuction()` is incorrect.

## Impact

This causes auctions to run for less time than intended if the first bid comes at a much later time after the auction was created. A shorter auction time could potentially allow fewer bids and cause it to be not economically efficient for the lenders, thus causing a suboptimal credit of liquidation funds to them.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/interfaces/IAuctionHouse.sol#L12-L13
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L80
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L166-L176

## Tool used

Manual Review

## Recommendation

`createAuction()` should initialize `firstBidTime = 0`.