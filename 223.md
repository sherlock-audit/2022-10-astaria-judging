hansfriese

high

# `AuctionHouse.createBid()` doesn't handle incoming payments properly.

## Summary
`AuctionHouse.createBid()` doesn't handle incoming payments properly.

## Vulnerability Detail
`AuctionHouse.createBid()` is used to receive bids from users and it refunds the last bidder when there is a new bidder with a higher bid amount.

But it doesn't handle the incoming payment for the new bidder [here](https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L118).

```solidity
    if (firstBidTime == 0) {
      auctions[tokenId].firstBidTime = block.timestamp.safeCastTo64();
    } else if (lastBidder != address(0)) {
      uint256 lastBidderRefund = amount - vaultPayment;
      _handleOutGoingPayment(lastBidder, lastBidderRefund);
    }

    _handleIncomingPayment(tokenId, vaultPayment, address(msg.sender)); //@audit vaultPayment => amount
```

It should request the whole `amount` but `amount - currentBid` now.

## Impact
`AuctionHouse.createBid()` requests a smaller amount than it should from the second bidder so the auction house protocol is lack funds.

## Code Snippet
https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L118

## Tool used
Manual Review

## Recommendation
It should request the whole amount like below.

```solidity
    _handleIncomingPayment(tokenId, amount, address(msg.sender));
```