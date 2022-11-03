0xRajeev

high

# Public vaults can become insolvent because of missing `yIntercept` update

## Summary

The deduction of `yIntercept` during payments is missing in `beforePayment()` which can lead to vault insolvency.

## Vulnerability Detail

`yIntercept` is declared as "sum of all LienToken amounts" and documented elsewhere as "yIntercept (virtual assets) of a PublicVault". It is used to calculate the total assets of a public vault as: `slope.mulDivDown(delta_t, 1) + yIntercept`.

It is expected to be updated on deposits, payments, withdrawals, liquidations. However, the deduction of `yIntercept` during payments is missing in `beforePayment()`. As noted in the function's Natspec:
```solidity
 /**
   * @notice Hook to update the slope and yIntercept of the PublicVault on payment.
   * The rate for the LienToken is subtracted from the total slope of the PublicVault, and recalculated in afterPayment().
   * @param lienId The ID of the lien.
   * @param amount The amount paid off to deduct from the yIntercept of the PublicVault.
   */
```
the amount of payment should be deducted from `yIntercept` but is missing. 

## Impact

PoC: https://gist.github.com/berndartmueller/477cc1026d3fe3e226795a34bb8a903a

This missing update will inflate the inferred value of the public vault corresponding to its actual value leading to eventual insolvency because of resulting protocol miscalculations.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L427-L442

## Tool used

Manual Review

## Recommendation

Update `yIntercept` in `beforePayment()` by the `amount` value.