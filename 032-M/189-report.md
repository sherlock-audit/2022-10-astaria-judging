0xRajeev

medium

# Incorrect `LienToken.changeInSlope` calculation can lead to vault insolvency

## Summary

The calculation of `newSlope` in `changeInSlope()` is incorrect which can lead to vault insolvency.

## Vulnerability Detail

Contrary to the `changeInSlope` function, the `calculateSlope` function uses the `_getOwed` function to calculate the owed amount (incl. the interest). The interest is calculated with the `_getInterest` function. This function takes care of the `lien.last` and also if a lien is expired already. This logic is completely missing in the `changeInSlope` for the `newSlope` calculation which makes it incorrect. Also, very importantly, in the `changeInSlope` function, the `INTEREST_DENOMINATOR` is missing which makes the value inflated causing an underflow error in the last line of `changeInSlope` function: `slope = oldSlope - newSlope`;. `oldslope`, which accounts for the `INTEREST_DENOMINATOR`, is less than `newSlope`.

## Impact

The incorrect `changeInSlope()` calculation can lead to reverts and vault insolvency because users cannot determine the implicit value of vaults while interacting with it as borrowers or lenders.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L453-L469
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L440-L445

## Tool used

Manual Review

## Recommendation

Make `changeInSlope()` consistent with `calculateSlope()` by implementing a separate helper function to calculate the interest accounting for all the parameters and reusing it in both places.