obront

high

# timeToEpochEnd calculates backwards, breaking protocol math

## Summary

When a lien is liquidated, it calls `timeToEpochEnd()` to determine if a liquidation accountant should be deployed and we should adjust the protocol math to expect payment in a future epoch. Because of an error in the implementation, all liquidations that will pay out in the current epoch are set up as future epoch liquidations.

## Vulnerability Detail

The `liquidate()` function performs the following check to determine if it should set up the liquidation to be paid out in a future epoch:

```solidity
if (PublicVault(owner).timeToEpochEnd() <= COLLATERAL_TOKEN.auctionWindow())
```

This check expects that `timeToEpochEnd()` will return the time until the epoch is over. However, the implementation gets this backwards:

```solidity
function timeToEpochEnd() public view returns (uint256) {
  uint256 epochEnd = START() + ((currentEpoch + 1) * EPOCH_LENGTH());

  if (epochEnd >= block.timestamp) {
    return uint256(0);
  }

  return block.timestamp - epochEnd;
}
```
If `epochEnd >= block.timestamp`, that means that there IS remaining time in the epoch, and it should perform the calculation to return `epochEnd - block.timestamp`. In the opposite case, where `epochEnd <= block.timestamp`, it should return zero.

The result is that the function returns 0 for any epoch that isn't over. Since `0 < COLLATERAL_TOKEN.auctionWindow())`, all liquidated liens will trigger a liquidation accountant and the rest of the accounting for future epoch withdrawals.

## Impact

Accounting for a future epoch withdrawal causes a number of inconsistencies in the protocol's math, the impact of which vary depending on the situation. As a few examples:
- It calls `decreaseEpochLienCount()`. This has the effect of artificially lowering the number of liens in the epoch, which will cause the final liens paid off in the epoch to revert (and will let us process the epoch earlier than intended).
- It sets the payee of the lien to the liquidation accountant, which will pay out according to the withdrawal ratio (whereas all funds should be staying in the vault).
- It calls `increaseLiquidationsExpectedAtBoundary()`, which can throw off the math when processing the epoch.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L562-L570

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L388-L415

## Tool used

Manual Review

## Recommendation

Fix the `timeToEpochEnd()` function so it calculates the remaining time properly:

```solidity
function timeToEpochEnd() public view returns (uint256) {
  uint256 epochEnd = START() + ((currentEpoch + 1) * EPOCH_LENGTH());

  if (epochEnd <= block.timestamp) {
    return uint256(0);
  }

  return epochEnd - block.timestamp; //
}
```