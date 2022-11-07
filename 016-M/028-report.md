obront

high

# _payment() function transfers full paymentAmount, overpaying first liens

## Summary

The `_payment()` function sends the full `paymentAmount` argument to the lien owner, which both (a) overpays lien owners if borrowers accidentally overpay and (b) sends the first lien owner all the funds for the entire loop of a borrower is intending to pay back multiple loans.

## Vulnerability Detail

There are two `makePayment()` functions in LienToken.sol. One that allows the user to specific a `position` (which specific lien they want to pay back, and another that iterates through their liens, paying each back.

In both cases, the functions call out to `_payment()` with a `paymentAmount`, which is sent (in full) to the lien owner.

```solidity
TRANSFER_PROXY.tokenTransferFrom(WETH, payer, payee, paymentAmount);
```

This behavior can cause problems in both cases.

The first case is less severe: If the user is intending to pay off one lien, and they enter a `paymentAmount` greater than the amount owed, the function will send the full `paymentAmount` to the lien owner, rather than just sending the amount owed.

The second case is much more severe: If the user is intending to pay towards all their loans, the `_makePayment()` function loops through open liens and performs the following:

```solidity
uint256 paymentAmount = totalCapitalAvailable;
for (uint256 i = 0; i < openLiens.length; ++i) {
  uint256 capitalSpent = _payment(
    collateralId,
    uint8(i),
    paymentAmount,
    address(msg.sender)
  );
  paymentAmount -= capitalSpent;
}
```

The `_payment()` function is called with the first lien with `paymentAmount` set to the full amount sent to the function. The result is that this full amount is sent to the first lien holder, which could greatly exceed the amount they are owed. 

## Impact

A user who is intending to pay off all their loans will end up paying all the funds they offered, but only paying off their first lien, potentially losing a large amount of funds.

## Code Snippet

The `_payment()` function with the core error:

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L594-L649

The `_makePayment()` function that uses `_payment()` and would cause the most severe harm:

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L410-L424

## Tool used

Manual Review

## Recommendation

In `_payment()`, if `lien.amount < paymentAmount`, set `paymentAmount = lien.amount`. 

The result will be that, in this case, only `lien.amount` is transferred to the lien owner, and this value is also returned from the function to accurately represent the amount that was paid.