0xRajeev

high

# Purchaser of a lien token may not receive payments

## Summary

A purchaser who buys out an existing lien via `buyoutLien()` will not receive future payments made to that lien holder if the seller had changed the lien payee via `setPayee()` and if they do not change it themselves after buying.

## Vulnerability Detail

`buyoutLien()` does not reset `lienData[lienId].payee` to either 0 or to the new owner. While the ownership is transferred, the payments made in `_payment()` get their payee via `getPayee()` which returns the owner only if the `lienData[lienId].payee` is set to the zero address but returns `lienData[lienId].payee` otherwise. This will still have the value set by the previous owner who will continue to receive the payments.

## Impact

The purchaser who buys out an existing lien via `buyoutLien()` will not receive future payments if the previous owner had set the payee but they do not change it via `setPayee()` themselves after buying. Given that `setPayee()` is considered optional (nothing specified in spec/docs/comments) and this reset is not part of the default flow, this will lead to a loss of purchaser's anticipated payments until they realize and reset the lien payee.

**Exploit Scenario:** A malicious lien seller can use `setPayee()` to set the payee to their address of choice and continue to receive payments, even after selling, if the buyer does not realize that they have to change the payee address to themselves after buying.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L619-L645
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L666-L676
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L678-L698

## Tool used

Manual Review

## Recommendation

`buyoutLien()` should reset `lienData[lienId].payee` to either the zero address or to the new owner.