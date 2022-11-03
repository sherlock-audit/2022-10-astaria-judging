0xRajeev

high

# Anyone can create a public vault

## Summary

The creation of new public vaults is missing the enforcement of access control to only allow whitelisted strategists to do so. This allows anyone to create a public vault and accept LP funds for ineffective/inefficient collateral/terms leading to the loss of their funds.

## Vulnerability Detail

Only whitelisted strategists should be allowed to deploy PublicVaults (per documentation) that accept funds from other liquidity providers. However, there is no such access control implemented, which allows malicious entities to deploy public vaults with ineffective/inefficient collateral/terms that lead to loss of LP funds.

## Impact

A malicious strategist can create a public vault which allows low quality NFTs as collateral, unreasonable loan terms etc. to overwhelmingly favor borrowers which may lead gullible lenders to lose their capital to non-repayments and liquidations.

## Code Snippet

1. Docs: https://docs.astaria.xyz/docs/threeactors
2. Docs: https://docs.astaria.xyz/docs/smart-contracts/PublicVault
3. Code: https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L284-L294

## Tool used

Manual Review

## Recommendation

Implement access control on `newPublicVault` to enforce whitelisting.