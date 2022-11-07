0xRajeev

high

# `LienToken.buyoutLien` will always revert

## Summary

`buyoutLien()` will always revert, preventing the borrower from refinancing.

## Vulnerability Detail

`buyoutFeeDenominator` is `0` without a setter which will cause `getBuyoutFee()` to revert in the `buyoutLien()` flow. 

## Impact

Refinancing is a crucial feature of the protocol to allow a borrower to refinance their loan if a certain minimum improvement of interest rate or duration is offered. The reverting `buyoutLien()` flow will prevent the borrower from refinancing and effectively lead to loss of their funds due to lock-in into currently held loans when better terms are available.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L71
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L456
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L377
4. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L132

## Tool used

Manual Review

## Recommendation

Initialize the buyout fee numerator and denominator in `AstariaRouter` and add their setters to `file()`.