0xRajeev

high

# Canceling an auction does not refund the current highest bidder

## Summary

If the collateral token owner cancels the active auction and repays outstanding debt, the current highest bidder will not be refunded and loses their funds.

## Vulnerability Detail

The `AuctionHouse.createBid()` function refunds the previous bidder if there is one. The same logic would also be necessary in the `AuctionHouse.cancelAuction()` function but is missing.

## Impact
If the collateral token owner cancels the active auction and repays outstanding debt (`reservePrice`), the current highest bidder will not be refunded and will therefore lose their funds which can also be exploited by a malicious borrower.

Potential exploit scenario: A malicious borrower can let the loan expire without repayment, trigger an auction, let bids below reserve price, and (hope to) front-run any bid >= reserve price to cancel the auction which effectively lets the highest bidder pay out (most of) the liens instead of the borrower.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L113-L116
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L210-L224

## Tool used

Manual Review

## Recommendation

Add the refund logic (via `_handleOutGoingPayment()` to the current bidder) in the cancel auction flow similar to the create bid auction flow.