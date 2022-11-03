0xRajeev

high

# A payment made towards multiple liens causes the borrower to lose funds to the payee

## Summary

A payment made towards multiple liens is entirely consumed for the first one causing the borrower to lose funds to the payee.

## Vulnerability Detail

A borrower can make a bulk payment against multiple liens for a collateral hoping to pay more than one at a time using `makePayment (uint256 collateralId, uint256 paymentAmount)` where the underlying `_makePayment()` loops over the open liens attempting to pay off more than one depending on the `totalCapitalAvailable` provided.

However, the entire `totalCapitalAvailable` is provided via `paymentAmount` in the call to `_payment()` in the first iteration which transfers that completely to the payee in its logic even if it exceeds that `lien.amount`. That total amount is returned as `capitalSpent` which makes the `paymentAmount` for next iteration equal to `0`.

## Impact

Only the first lien is paid off and the entire payment is sent to its payee. The remaining liens remain unpaid. The payment maker (i.e. borrower ) loses funds to the payee.

## Code Snippet
1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L387-L389
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L410-L424
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L630-L645

## Tool used

Manual Review

## Recommendation

Add `paymentAmount -= lien.amount` in the `else` block of `_payment()`.