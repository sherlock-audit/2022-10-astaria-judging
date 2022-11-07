0xRajeev

medium

# Extension logic incorrectly extends the auction by an additional amount of existing duration

## Summary

Incorrect auction extension logic extends the auction by an additional amount of the previous duration instead of extending it by 15 minutes.

## Vulnerability Detail

The calculation of `newDuration` incorrectly adds `duration` in the auction extension logic. This causes the new duration to be extended by an additional amount of the existing duration, instead of an additional 15 minutes (`timeBuffer`), when a bid is created in the last 15 mins of the existing auction duration.

## Impact

Delayed payout of funds/collateral upon auction completion only after the newly extended duration.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L135-L137

## Tool used

Manual Review

## Recommendation

Change the calculation to `uint64 newDuration = uint256(block.timestamp + timeBuffer - firstBidTime).safeCastTo64();`