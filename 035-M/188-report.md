0xRajeev

high

# `LiquidationAccountant.claim()` can be called by anyone causing vault insolvency

## Summary

`LiquidationAccountant.claim()` can be called by anyone to reduce the implied value of a public vault.

## Vulnerability Detail

`LiquidationAccountant.claim()` is called by the `PublicVault` as part of the `processEpoch()` flow. But it has no access control and can be called by anyone and any number of times. If called after `finalAuctionEnd`, one will be able to trigger `decreaseYIntercept()` on the vault even if they cannot affect fund transfer to withdrawing liquidity providers and the PublicVault.

## Impact

This allows anyone to manipulate the `yIntercept` of a public vault by triggering the `claim()` flow after liquidations resulting in vault insolvency.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LiquidationAccountant.sol#L65-L97

## Tool used

Manual Review

## Recommendation

Allow only vault to call `claim()` by requiring authorizations.