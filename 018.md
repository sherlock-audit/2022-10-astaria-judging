obront

medium

# Bids cannot be created within timeBuffer of completion of a max duration auction

## Summary

The auction mechanism is intended to watch for bids within `timeBuffer` of the end of the auction, and automatically increase the remaining duration to `timeBuffer` if such a bid comes in.

There is an error in the implementation that causes all bids within `timeBuffer` of the end of a max duration auction to revert, effectively ending the auction early and cutting off bidders who intended to wait until the end.

## Vulnerability Detail

In the `createBid()` function in AuctionHouse.sol, the function checks if a bid is within the final `timeBuffer` of the auction:

```solidity
if (firstBidTime + duration - block.timestamp < timeBuffer)
```

If so, it sets `newDuration` to equal the amount that will extend the auction to `timeBuffer` from now:

```solidity
uint64 newDuration = uint256( duration + (block.timestamp + timeBuffer - firstBidTime) ).safeCastTo64();
```

If this `newDuration` doesn't extend beyond the `maxDuration`, this works great. However, if it does extend beyond `maxDuration`, the following code is used to update `duration`:

```solidity
auctions[tokenId].duration = auctions[tokenId].maxDuration - firstBidTime;
```

This code is incorrect. `maxDuration` will be a duration for the contest (currently set to 3 days), whereas `firstTimeBid` is a timestamp for the start of the auction (current timestamps are > 1 billion). 

Subtracting `firstTimeBid` from `maxDuration` will underflow, which will revert the function.

## Impact

- Bidders who expected to wait until the end of the auction to vote will be cut off from voting, as the auction will revert their bids.
- Vaults whose collateral is up for auction will earn less than they otherwise would have.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L127-L146

## Tool used

Manual Review

## Recommendation

Change this assignment to simply assign `duration` to `maxDuration`, as follows:

```solidity
auctions[tokenId].duration = auctions[tokenId].maxDuration
```