0xRajeev

high

# `VaultImplementation._validateCommitment` may prevent liens that satisfy their terms of `maxPotentialDebt`

## Summary

The calculation of `potentialDebt` in `VaultImplementation._validateCommitment()` is incorrect and will cause a DoS to legitimate borrowers.

## Vulnerability Detail

The calculation of potentialDebt in `VaultImplementation._validateCommitment()` is incorrect because it computes `uint256 potentialDebt = seniorDebt * (ld.rate + 1) * ld.duration;` which incorrectly adds a factor of `ld.duration` to `seniorDebt` thus making the potential debt much higher by that factor than it will be. The use of `INTEREST_DENOMINATOR` and implied lien rate is also missing here. 


## Impact

Liens that would have otherwise satisfied the constraint of `potentialDebt <= ld.maxPotentialDebt` will fail because of this miscalculation and will cause a DoS to legitimate borrowers and likely all of them.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L221-L225
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L256-L262

## Tool used

Manual Review

## Recommendation

Change the calculation to `uint256 potentialDebt = seniorDebt * (ld.rate * ld.duration + 1).mulDivDown(1, INTEREST_DENOMINATOR);`. This should also consider the implied rate of all the liens against the collateral instead of only this lien.