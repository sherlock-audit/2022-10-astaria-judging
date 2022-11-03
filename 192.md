0xRajeev

medium

# `LienToken.createLien` may prevent liens that satisfy their terms of `maxPotentialDebt`

## Summary

The `potentialDebt` calculation in `createLien` is incorrect.

## Vulnerability Detail

The calculated `potentialDebt` is effectively `(impliedRate + totalDebt) * params.terms.duration` because `getImpliedRate()` returns `impliedRate = impliedRate.mulDivDown(1, totalDebt);`. The calculated `potentialDebt` because of multiplying `totalDebt` by duration is significantly higher than it actually is and so will fail the `params.terms.maxPotentialDebt` check and revert. 

## Impact

This will cause DoS on valid lien creation.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L256-L260
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L526

## Tool used

Manual Review

## Recommendation

Have `getImpliedRate()` *not* do `impliedRate = impliedRate.mulDivDown(1, totalDebt);` and calculate `potentialDebt` as `totalDebt * (1 + impliedRate *  params.terms.duration * mulDivDown(1, INTEREST_DENOMINATOR)`.