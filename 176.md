0xRajeev

high

# Anyone can deposit and mint withdrawal proxy shares to capture distributed yield from borrower interests

## Summary

Anyone can deposit and mint Withdrawal proxy shares by directly interacting with the base `ERC4626Cloned` contract's functions, allowing them to capture distributed yield from borrower interests.

## Vulnerability Detail

The `WithdrawProxy` contract extends the `ERC4626Cloned` vault contract implementation. The `ERC4626Cloned` contract has the functionality to deposit and mint vault shares. Usually, withdrawal proxy shares are only distributed via the `WithdrawProxy.mint` function, which is only called by the `PublicVault.redeemFutureEpoch `function. Anyone can deposit WETH into a deployed Withdraw proxy to receive shares, wait until assets (WETH) are deposited via the `PublicVault.transferWithdrawReserve` or `LiquidationAccountant.claim` function and then redeem their shares for WETH assets. 

## Impact

By depositing/minting directly to the Withdraw proxy, one can get interest yield on-demand without being an LP and having capital locked for epoch(s). This may potentially be timed in a way to deposit/mint only when we know that interest yields are being paid by a borrower who is not defaulting on their loan. The returns are diluted for the LPs at the expense of someone who directly interacts with the underlying proxy.
 
## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/ERC4626-Cloned.sol#L305-L339
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L198

## Tool used

Manual Review

## Recommendation

Overwrite the `ERC4626Cloned.afterDeposit` function and revert to prevent public deposits and mints.