obront

high

# isValidRefinance will approve invalid refinances and reject valid refinances due to buggy math

## Summary

The math in `isValidRefinance()` checks whether the rate increased rather than decreased, resulting in invalid refinances being approved and valid refinances being rejected.

## Vulnerability Detail

When trying to buy out a lien from `LienToken.sol:buyoutLien()`, the function calls `AstariaRouter.sol:isValidRefinance()` to check whether the refi terms are valid.

```solidity
if (!ASTARIA_ROUTER.isValidRefinance(lienData[lienId], ld)) {
  revert InvalidRefinance();
}
```
One of the roles of this function is to check whether the rate decreased by more than 0.5%. From the docs:

> An improvement in terms is considered if either of these conditions is met:
> - The loan interest rate decrease by more than 0.5%.
> - The loan duration increases by more than 14 days.

The current implementation of the function does the opposite. It calculates a `minNewRate` (which should be `maxNewRate`) and then checks whether the new rate is greater than that value.

```solidity
uint256 minNewRate = uint256(lien.rate) - minInterestBPS;
return (newLien.rate >= minNewRate ...
```

The result is that if the new rate has increased (or decreased by less than 0.5%), it will be considered valid, but if it has decreased by more than 0.5% (the ideal behavior) it will be rejected as invalid.

## Impact

- Users can perform invalid refinances with the wrong parameters.
- Users who should be able to perform refinances at better rates will not be able to.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L482-L491

## Tool used

Manual Review

## Recommendation

Flip the logic used to check the rate to the following:

```solidity
uint256 maxNewRate = uint256(lien.rate) - minInterestBPS;
return (newLien.rate <= maxNewRate...
```