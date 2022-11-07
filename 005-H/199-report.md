0xRajeev

high

# Canceling an auction with 0 bids will only partially pay back the outstanding debt

## Summary

Canceling an auction with 0 bids only partially pays back the outstanding debt without removing outstanding liens resulting in the collateralized NFT being locked forever.

## Vulnerability Detail

Given this scenario of an auction with 0 bids and, for example, a `reservePrice` of 10 ETH and `initiatorFee` set to 10 (= 10%), with the following sequence executed:

1. The auction gets canceled
2. In `AuctionHouse.cancelAuction()`, `_handleIncomingPayment()` accepts incoming transfer of 10 ETH reserve price
3. In `_handleIncomingPayment()`, the `initiatorFee` of 10% is calculated, deducted from the 10 ETH to transfer 1 ETH to the initiator
4. The remaining 9 ETH is then used to pay back the outstanding liens (open debt)

However, the outstanding debt was initially 10 ETH (= `reservePrice`). After deducting the initiator fee, only 9 ETH remains. This means that the debt is not fully paid back. But the auction will successfully be cancelled but with outstanding debt remaining. There is also no explicit removal of liens (as there is in `endAuction`).

## Impact

A borrower canceling an auction with the expected reserve price will leave behind existing unpaid liens that will make the `releaseCheck` modifier applied to `releaseToAddress()` prevent the borrower from releasing their NFT collateral successfully even after paying the reserve price during auction cancellation. The borrower's collateralized NFT is locked forever despite paying the outstanding lien amount during the auction cancellation.


## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L210-L224
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L253-L276
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L202
4. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/CollateralToken.sol#L135-L142

## Tool used

Manual Review

## Recommendation
The auction cancellation amount required should be reserve price + liquidation fee. On payment, remaining liens should be removed.