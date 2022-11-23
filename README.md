# Issue H-1: The implied value of a public vault can be impaired, liquidity providers can lose funds 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/272 

## Found by 
Jeiwan

## Summary
The implied value of a public vault can be impaired, liquidity providers can lose funds
## Vulnerability Detail
Borrowers can partially repay their liens, which is handled by the `_payment` function ([LienToken.sol#L594](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L594)).
When repaying a part of a lien, `lien.amount` is updated to include currently accrued debt ([LienToken.sol#L605-L617](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L605-L617)):
```solidity
Lien storage lien = lienData[lienId];
lien.amount = _getOwed(lien); // @audit current debt, including accrued interest; saved to storage!
```
Notice that `lien.amount` is updated in storage, and `lien.last` wasn't updated.

Then, lien's slope is subtracted from vault's slope accumulator to be re-calculated after the repayment ([LienToken.sol#L620-L630](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L620-L630)):
```solidity
if (isPublicVault) {
  // @audit calculates and subtracts lien's slope from vault's slope
  IPublicVault(lienOwner).beforePayment(lienId, paymentAmount);
}
if (lien.amount > paymentAmount) {
  lien.amount -= paymentAmount;
  // @audit lien.last is updated only after payment amount subtraction
  lien.last = block.timestamp.safeCastTo32();
  // slope does not need to be updated if paying off the rest, since we neutralize slope in beforePayment()
  if (isPublicVault) {
    // @audit re-calculates and re-applies lien's slope after the repayment
    IPublicVault(lienOwner).afterPayment(lienId);
  }
}
```

In the `beforePayment` function, `LIEN_TOKEN().calculateSlope(lienId)` is called to calculate lien's current slope ([PublicVault.sol#L433-L442](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L433-L442)):
```solidity
function beforePayment(uint256 lienId, uint256 amount) public onlyLienToken {
  _handleStrategistInterestReward(lienId, amount);
  uint256 lienSlope = LIEN_TOKEN().calculateSlope(lienId);
  if (lienSlope > slope) {
    slope = 0;
  } else {
    slope -= lienSlope;
  }
  last = block.timestamp;
}
```

The `calculateSlope` function reads a lien from storage and calls `_getOwed` again ([LienToken.sol#L440-L445](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L440-L445)):
```solidity
function calculateSlope(uint256 lienId) public view returns (uint256) {
  // @audit lien.amount includes interest accrued so far
  Lien memory lien = lienData[lienId];
  uint256 end = (lien.start + lien.duration);
  uint256 owedAtEnd = _getOwed(lien, end);
  // @audit lien.last wasn't updated in `_payment`, it's an older timestamp
  return (owedAtEnd - lien.amount).mulDivDown(1, end - lien.last);
}
```

This is where double counting of accrued interest happens. Recall that lien's amount already includes the interest that was accrued by this moment (in the `_payment` function). Now, interest is calculated again and *is applied to the amount that already includes (a portion) it* ([LienToken.sol#L544-L550](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L544-L550)):
```solidity
function _getOwed(Lien memory lien, uint256 timestamp)
  internal
  view
  returns (uint256)
{
  // @audit lien.amount already includes interest accrued so far
  return lien.amount + _getInterest(lien, timestamp);
}
```

[LienToken.sol#L177-L196](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L177-L196):
```solidity
function _getInterest(Lien memory lien, uint256 timestamp)
  internal
  view
  returns (uint256)
{
  if (!lien.active) {
    return uint256(0);
  }
  uint256 delta_t;
  if (block.timestamp >= lien.start + lien.duration) {
    delta_t = uint256(lien.start + lien.duration - lien.last);
  } else {
    // @audit lien.last wasn't updated in `_payment`, so the `delta_t` is bigger here
    delta_t = uint256(timestamp.safeCastTo32() - lien.last);
  }
  return
    // @audit rate applied to a longer delta_t and multiplied by a bigger amount than expected
    delta_t.mulDivDown(lien.rate, 1).mulDivDown(
      lien.amount,
      INTEREST_DENOMINATOR
    );
}
```
## Impact
Double counting of interest will result in a wrong lien slope, which will affect the vault's slope accumulator.  This will result in an invalid implied value of a vault ([PublicVault.sol#L406-L413](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L406-L413)):
1. If miscalculated lien slope is bigger than expected, vault's slope will be smaller than expected (due to the subtraction in `beforePayment`), and vault's implied value will also be smaller. Liquidity providers will lose money because they won't be able to redeem the whole liquidity (vault's implied value, `totalAssets`, is used in the conversion of LP shares, [ERC4626-Cloned.sol#L392-L412](https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/ERC4626-Cloned.sol#L392-L412))
1. If miscalculated lien slope is smaller than expected, vault's slope will be higher, and vaults implied value will also be higher. However, it won't be backed by actual liquidity, thus the liquidity providers that exit earlier will get a bigger share of the underlying assets. The last liquidity provider won't be able to get their entire share.

## Code Snippet
See Vulnerability Detail
## Tool used
Manual Review
## Recommendation
In the `_payment` function, consider updating `lien.amount` after the `beforePayment` call:
```diff
--- a/src/LienToken.sol
+++ b/src/LienToken.sol
@@ -614,12 +614,13 @@ contract LienToken is ERC721, ILienToken, Auth, TransferAgent {
       type(IPublicVault).interfaceId
     );

-    lien.amount = _getOwed(lien);
-
     address payee = getPayee(lienId);
     if (isPublicVault) {
       IPublicVault(lienOwner).beforePayment(lienId, paymentAmount);
     }
+
+    lien.amount = _getOwed(lien);
+
     if (lien.amount > paymentAmount) {
       lien.amount -= paymentAmount;
       lien.last = block.timestamp.safeCastTo32();
```
In this case, lien's slope calculation won't be affected in the `beforePayment` call and the correct slope will be removed from the slope accumulator.

# Issue H-2: `LIEN_TOKEN.ownerOf(i)` should be `LIEN_TOKEN.ownerOf(liensRemaining[i])` 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/259 

## Found by 
\_\_141345\_\_, 0xRajeev

## Summary

In `endAuction()`, the check for public vault owner is referred to the wrong lien token id. And the actual vault lien amount is not properly recorded. 


## Vulnerability Detail

The lien token id should be queried is `liensRemaining[i]` instead of `i`.


## Impact

`YIntercept` will not be correctly recorded. The accounting for LienToken amounts will be wrong. Hence the `totalAssets` on book will be wrong, eventually the contract and users could lose fund due to the wrong accounting.



## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L192-L204


## Tool used

Manual Review

## Recommendation

Change `LIEN_TOKEN.ownerOf(i)` to `LIEN_TOKEN.ownerOf(liensRemaining[i])`.

# Issue H-3: buyoutLien() will cause the vault to fail to processEpoch() 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/245 

## Found by 
bin2chen

## Summary
LienToken#buyoutLien() did not reduce vault#liensOpenForEpoch
when vault#processEpoch()will check vault#liensOpenForEpoch[currentEpoch] == uint256(0)
so processEpoch() will fail

## Vulnerability Detail
when create LienToken , vault#liensOpenForEpoch[currentEpoch] will ++
when repay  or liquidate ,  vault#liensOpenForEpoch[currentEpoch] will --
and LienToken#buyoutLien() will transfer from  vault to to other receiver,so liensOpenForEpoch need reduce 
```solidity
function buyoutLien(ILienToken.LienActionBuyout calldata params) external {
   ....
    /**** tranfer but not liensOpenForEpoch-- *****/
    _transfer(ownerOf(lienId), address(params.receiver), lienId);
  }
```


## Impact
processEpoch() maybe fail

## Code Snippet
https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L121

## Tool used

Manual Review

## Recommendation

```solidity
  function buyoutLien(ILienToken.LienActionBuyout calldata params) external {
....

+   //do decreaseEpochLienCount()
+   address lienOwner = ownerOf(lienId);
+    bool isPublicVault = IPublicVault(lienOwner).supportsInterface(
+      type(IPublicVault).interfaceId
+    );
+    if (isPublicVault && !AUCTION_HOUSE.auctionExists(collateralId)) {      
+        IPublicVault(lienOwner).decreaseEpochLienCount(
+          IPublicVault(lienOwner).getLienEpoch(lienData[lienId].start + lienData[lienId].duration)
+        );
+    }    

    lienData[lienId].last = block.timestamp.safeCastTo32();
    lienData[lienId].start = block.timestamp.safeCastTo32();
    lienData[lienId].rate = ld.rate.safeCastTo240();
    lienData[lienId].duration = ld.duration.safeCastTo32();
    _transfer(ownerOf(lienId), address(params.receiver), lienId);
  }
```

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

Duplicate of #194, two sides of the same coin. One points out it that buyout doesn't decrement correctly on one side and the other points out it doesn't increment correctly on the other side

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Duplicate of #194, two sides of the same coin. One points out it that buyout doesn't decrement correctly on one side and the other points out it doesn't increment correctly on the other side

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation rejected.

The argument is not giving us full conviction this should be tagged as a duplicate.

**sherlock-admin**

> Escalation rejected.
> 
> The argument is not giving us full conviction this should be tagged as a duplicate.

This issue's escalations have been rejected!

Watsons who escalated this issue will have their escalation amount deducted from their next payout.



# Issue H-4: _deleteLienPosition can be called by anyone to delete any lien they wish 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/233 

## Found by 
tives, ctf\_sec, 0x0, obront, TurnipBoy, yixxas, zzykxx

## Summary

`_deleteLienPosition` is a public function that doesn't check the caller. This allows anyone to call it an remove whatever lien they wish from whatever collateral they wish

## Vulnerability Detail

    function _deleteLienPosition(uint256 collateralId, uint256 position) public {
      uint256[] storage stack = liens[collateralId];
      require(position < stack.length, "index out of bounds");

      emit RemoveLien(
        stack[position],
        lienData[stack[position]].collateralId,
        lienData[stack[position]].position
      );
      for (uint256 i = position; i < stack.length - 1; i++) {
        stack[i] = stack[i + 1];
      }
      stack.pop();
    }

`_deleteLienPosition` is a `public` function and doesn't validate that it's being called by any permissioned account. The result is that anyone can call it to delete any lien that they want. It wouldn't remove the lien data but it would remove it from the array associated with `collateralId`, which would allow it to pass the `CollateralToken.sol#releaseCheck` and the underlying to be withdrawn by the user. 

## Impact

All liens can be deleted completely rugging lenders

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L651-L664

## Tool used

Manual Review

## Recommendation

Change `_deleteLienPosition` to `internal` rather than `public`.

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

Deleting leans allows borrower to steal funds because they never have to repay the funds the borrowed. Should be high risk

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Deleting leans allows borrower to steal funds because they never have to repay the funds the borrowed. Should be high risk

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted

**sherlock-admin**

> Escalation accepted

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue H-5: `commitToLiens` always reverts 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/204 

## Found by 
bin2chen, HonorLt, 0xRajeev, rvierdiiev, Jeiwan

## Summary

The function `commitToLiens()` always reverts at the call to `_returnCollateral()` which prevents borrowers from depositing collateral and requesting loans in the protocol.

## Vulnerability Detail

The collateral token with `collateralId` is already minted directly to the caller (i.e. borrower) in `commitToLiens()` at the call to `_transferAndDepositAsset()` function. That's because while executing `_transferAndDepositAsset` the NFT is transferred to `COLLATERAL_TOKEN` whose `onERC721Received` mints the token with `collateralId` to borrower (`from` address) and not the `operator_` (i.e. `AstariaRouter`) because `operator_ != from_`.

However, the call to `_returnCollateral()` in `commitToLiens()` incorrectly assumes that this has been minted to the operator and attempts to transfer it to the borrower which will revert because the `collateralId` is not owned by  `AstariaRouter` as it has already been transferred/minted to the borrower.

## Impact

The function `commitToLiens()` always reverts, preventing borrowers from depositing collateral and requesting loans in the protocol, thereby failing to bootstrap its core NFT lending functionality.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L244-L274
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L578-L587
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/CollateralToken.sol#L282-L284
4. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L589-L591

## Tool used

Manual Review

## Recommendation
Remove the call to `_returnCollateral()` in `commitToLiens()`.

## Discussion

**secureum**

Escalate for 2 USDC.

Given the impact of failing to bootstrap core protocol functionality as described above, we still think this is of high-severity (not Medium as judged) unlike a DoS that affects only a minority of protocol flows. Also, this is not a dup of #195.

cc @berndartmueller @lucyoa

**sherlock-admin**

 > Escalate for 2 USDC.
> 
> Given the impact of failing to bootstrap core protocol functionality as described above, we still think this is of high-severity (not Medium as judged) unlike a DoS that affects only a minority of protocol flows. Also, this is not a dup of #195.
> 
> cc @berndartmueller @lucyoa

You've created a valid escalation for 2 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted

**sherlock-admin**

> Escalation accepted

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue H-6: Canceling an auction does not refund the current highest bidder 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/202 

## Found by 
chainNue, minhquanym, hansfriese, bin2chen, TurnipBoy, peanuts, neila, Prefix, csanuragjain, 0xRajeev, Jeiwan

## Summary

If the collateral token owner cancels the active auction and repays outstanding debt, the current highest bidder will not be refunded and loses their funds.

## Vulnerability Detail

The `AuctionHouse.createBid()` function refunds the previous bidder if there is one. The same logic would also be necessary in the `AuctionHouse.cancelAuction()` function but is missing.

## Impact
If the collateral token owner cancels the active auction and repays outstanding debt (`reservePrice`), the current highest bidder will not be refunded and will therefore lose their funds which can also be exploited by a malicious borrower.

Potential exploit scenario: A malicious borrower can let the loan expire without repayment, trigger an auction, let bids below reserve price, and (hope to) front-run any bid >= reserve price to cancel the auction which effectively lets the highest bidder pay out (most of) the liens instead of the borrower.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L113-L116
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L210-L224

## Tool used

Manual Review

## Recommendation

Add the refund logic (via `_handleOutGoingPayment()` to the current bidder) in the cancel auction flow similar to the create bid auction flow.

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

Huge loss of funds for bidder. High risk

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Huge loss of funds for bidder. High risk

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**


Escalation accepted.



**sherlock-admin**

> 
> Escalation accepted.
> 
> 

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue H-7: Canceling an auction with 0 bids will only partially pay back the outstanding debt 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/199 

## Found by 
0x4141, bin2chen, TurnipBoy, 0xRajeev, Jeiwan

## Summary

Canceling an auction with 0 bids only partially pays back the outstanding debt without removing outstanding liens resulting in the collateralized NFT being locked forever.

## Vulnerability Detail

Given this scenario of an auction with 0 bids and, for example, a `reservePrice` of 10 ETH and `initiatorFee` set to 10 (= 10%), with the following sequence executed:

1. The auction gets canceled
2. In `AuctionHouse.cancelAuction()`, `_handleIncomingPayment()` accepts incoming transfer of 10 ETH reserve price
3. In `_handleIncomingPayment()`, the `initiatorFee` of 10% is calculated, deducted from the 10 ETH to transfer 1 ETH to the initiator
4. The remaining 9 ETH is then used to pay back the outstanding liens (open debt)

However, the outstanding debt was initially 10 ETH (= `reservePrice`). After deducting the initiator fee, only 9 ETH remains. This means that the debt is not fully paid back. But the auction will successfully be cancelled but with outstanding debt remaining. There is also no explicit removal of liens (as there is in `endAuction`).

## Impact

A borrower canceling an auction with the expected reserve price will leave behind existing unpaid liens that will make the `releaseCheck` modifier applied to `releaseToAddress()` prevent the borrower from releasing their NFT collateral successfully even after paying the reserve price during auction cancellation. The borrower's collateralized NFT is locked forever despite paying the outstanding lien amount during the auction cancellation.


## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L210-L224
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L253-L276
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L202
4. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/CollateralToken.sol#L135-L142

## Tool used

Manual Review

## Recommendation
The auction cancellation amount required should be reserve price + liquidation fee. On payment, remaining liens should be removed.

# Issue H-8: Canceling an auction will result in a loss of borrower funds towards initiator fees 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/198 

## Found by 
0xRajeev

## Summary

The initiator fee on the current bid has already been accounted during that bid but is expected to be paid again on cancellation by the borrower.

## Vulnerability Detail

Consider an active auction with a reserve price of 10 ETH and a current bid of 9 ETH. If the auction gets canceled, the `transferAmount` will be 10 ETH. Once the cancellation logic adds repayment to current bidder (see other finding reported on this issue), the initiator fees for cancellation should only be calculated on the delta payment of 1 ETH because the fees for the earlier 9 ETH bid has already been paid to the initiator. However, this is not accounted and requires the borrower to pay the initiator fees on the entire reserve price leading to overpayment towards the initiator and loss of funds for the borrower.

## Impact

Canceling an auction will require paying (a lot) more initiator fees than needed resulting in a loss of funds for the borrower.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L265-L268
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L220

## Tool used

Manual Review

## Recommendation

The cancellation amount required should be the reserve price + liquidation fee, where the fee is calculated on `(reserve price - current bid)` and not the reserve price.

# Issue H-9: Public vaults can become insolvent because of missing `yIntercept` update 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/197 

## Found by 
0xRajeev, zzykxx

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

## Discussion

**androolloyd**

tagging @SantiagoGregory but i believe this is a documentation error, will address

**secureum**

Escalate for 2 USDC.

Given the vault insolvency impact as described and demonstrated by the PoC, we still think this is a high-severity impact (not Medium as judged). The other dup #92 also reported this as a High.

cc @berndartmueller @lucyoa

**sherlock-admin**

 > Escalate for 2 USDC.
> 
> Given the vault insolvency impact as described and demonstrated by the PoC, we still think this is a high-severity impact (not Medium as judged). The other dup #92 also reported this as a High.
> 
> cc @berndartmueller @lucyoa

You've created a valid escalation for 2 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted

**sherlock-admin**

> Escalation accepted

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue H-10: `LienToken.buyoutLien` will always revert 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/196 

## Found by 
8olidity, ctf\_sec, zzykxx, supernova, yixxas, neila, cccz, 0xRajeev, rvierdiiev

## Summary

`buyoutLien()` will always revert, preventing the borrower from refinancing.

## Vulnerability Detail

`buyoutFeeDenominator` is `0` without a setter which will cause `getBuyoutFee()` to revert in the `buyoutLien()` flow. 

## Impact

Refinancing is a crucial feature of the protocol to allow a borrower to refinance their loan if a certain minimum improvement of interest rate or duration is offered. The reverting `buyoutLien()` flow will prevent the borrower from refinancing and effectively lead to loss of their funds due to lock-in into currently held loans when better terms are available.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L71
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L456
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L377
4. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L132

## Tool used

Manual Review

## Recommendation

Initialize the buyout fee numerator and denominator in `AstariaRouter` and add their setters to `file()`.

## Discussion

**secureum**

Escalate for 2 USDC.

Given the potential impact to different flows/contexts, we still think this is a high-severity impact (not Medium as judged). A majority of the dups (while some are dups of a different but related issue) also reported this as a High.

cc @berndartmueller @lucyoa

**sherlock-admin**

 > Escalate for 2 USDC.
> 
> Given the potential impact to different flows/contexts, we still think this is a high-severity impact (not Medium as judged). A majority of the dups (while some are dups of a different but related issue) also reported this as a High.
> 
> cc @berndartmueller @lucyoa

You've created a valid escalation for 2 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted

**sherlock-admin**

> Escalation accepted

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue H-11: Lien count per epoch is not updated ultimately locking the collateralized NFT 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/194 

## Found by 
0xRajeev

## Summary

The lien count per epoch is not updated, causing payments to the lien and liquidation of the lien to revert.

## Vulnerability Detail

The PublicVault contract keeps track of open liens per epoch to prevent starting a new epoch with open liens. The lien count for a given epoch is decreased whenever a lien is fully paid back (either through a regular payment or a liquidation payment). However, if a lien is bought out, the lien start will be set to the current `block.timestamp` and the duration to the newly provided duration.

If the borrower now wants to make a payment to this lien, the `LienToken._payment` function will evaluate the lien's current epoch and will use a different epoch as when the lien was initially created. The attempt to then call `IPublicVault(lienOwner).decreaseEpochLienCount` will fail, as the lien count for the new epoch has not been increased yet. The same will happen for liquidations.

## Impact

After a lien buyout, payments to the lien and liquidating the lien will revert, which will ultimately lock the collateralized NFT.

This will certainly prevent the borrower from making payments towards that future-epoch lien in the current epoch because `decreaseEpochLienCount()` will revert. However, even after the epoch progresses to the next one via `processEpoch()`, the `liensOpenForEpoch` for the new epoch still does not account for the previously bought out lien aligned to this new epoch because `liensOpenForEpoch` is only updated in two places:
    1.  `_increaseOpenLiens()` which is not called by anyone 
    2. `_afterCommitToLien()` <- `commitToLien()` <-- <-- `commitToLiens()` which happens only for new lien commitments

This will prevent the borrower from making payments towards the previously bought lien that aligns to the current epoch, which will force a liquidation on exceeding lien duration. Depending on when the liquidation can be triggered, if this condition is satisfied `PublicVault(owner).timeToEpochEnd() <= COLLATERAL_TOKEN.auctionWindow()` then `decreaseEpochLienCount()` will revert to prevent auctioning and lock the borrower's collateral in the protocol.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L259-L262
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L634-L636
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L153
4. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L399

## Tool used

Manual Review

## Recommendation

`liensOpenForEpoch` should be incremented when a lien is bought with a duration spilling into an epoch higher than the current one.

# Issue H-12: Purchaser of a lien token may not receive payments 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/193 

## Found by 
obront, 0xRajeev, rvierdiiev

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

# Issue H-13: A payment made towards multiple liens causes the borrower to lose funds to the payee 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/190 

## Found by 
obront, 0xRajeev, zzykxx, Jeiwan

## Summary

A payment made towards multiple liens is entirely consumed for the first one causing the borrower to lose funds to the payee.

## Vulnerability Detail

A borrower can make a bulk payment against multiple liens for a collateral hoping to pay more than one at a time using `makePayment (uint256 collateralId, uint256 paymentAmount)` where the underlying `_makePayment()` loops over the open liens attempting to pay off more than one depending on the `totalCapitalAvailable` provided.

However, the entire `totalCapitalAvailable` is provided via `paymentAmount` in the call to `_payment()` in the first iteration which transfers that completely to the payee in its logic even if it exceeds that `lien.amount`. That total amount is returned as `capitalSpent` which makes the `paymentAmount` for next iteration equal to `0`.

## Impact

Only the first lien is paid off and the entire payment is sent to its payee. The remaining liens remain unpaid. The payment maker (i.e. borrower ) loses funds to the payee.

## Code Snippet
1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L387-L389
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L410-L424
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L630-L645

## Tool used

Manual Review

## Recommendation

Add `paymentAmount -= lien.amount` in the `else` block of `_payment()`.

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

Clear loss of funds. Should be high

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Clear loss of funds. Should be high

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted

**sherlock-admin**

> Escalation accepted

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue H-14: `LiquidationAccountant.claim()` can be called by anyone causing vault insolvency 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/188 

## Found by 
TurnipBoy, obront, 0xRajeev, rvierdiiev

## Summary

`LiquidationAccountant.claim()` can be called by anyone to reduce the implied value of a public vault.

## Vulnerability Detail

`LiquidationAccountant.claim()` is called by the `PublicVault` as part of the `processEpoch()` flow. But it has no access control and can be called by anyone and any number of times. If called after `finalAuctionEnd`, one will be able to trigger `decreaseYIntercept()` on the vault even if they cannot affect fund transfer to withdrawing liquidity providers and the PublicVault.

## Impact

This allows anyone to manipulate the `yIntercept` of a public vault by triggering the `claim()` flow after liquidations resulting in vault insolvency.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LiquidationAccountant.sol#L65-L97

## Tool used

Manual Review

## Recommendation

Allow only vault to call `claim()` by requiring authorizations.

## Discussion

**SantiagoGregory**

Our updated LiquidationAccountant implementation (now moved to WithdrawProxy) tracks a hasClaimed bool to make sure claim() is only called once (we also now block claim() from being called until after finalAuctionEnd).

**IAmTurnipBoy**

Escalate for 1 USDC

Abuse can cause vault to implode and cause loss of funds to all depositors. Should be high

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Abuse can cause vault to implode and cause loss of funds to all depositors. Should be high

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**


Escalation accepted.



**sherlock-admin**

> 
> Escalation accepted.
> 
> 

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue H-15: A malicious lien owner can exploit a reentrancy to steal LP funds 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/187 

## Found by 
0xRajeev

## Summary

A malicious lien owner can exploit a reentrancy in auctions to have their liens removed without making their payments and thus stealing LP funds.

## Vulnerability Detail

A malicious lien owner can exploit a reentrancy from the callback to `decreaseYIntercept()` in `endAuction()` to call `cancelAuction()` and then take a new loan whose lien token is removed when control returns back to `endAuction()`.

## Impact

PoC: https://gist.github.com/lucyoa/901a7713fded73293b5e4f9452344c5a

Exploit scenario sequence:
1. Strategist creates the public vault
2. Liquidity provider puts 50 ETH into vault
3. Malicious borrower takes a loan of 10 ETH by depositing NFT
5. Borrower buys out their own lien via `_buyoutLien` by paying 10ETH + fee
6. Borrower does not repay within lien duration to trigger liquidation
7. Borrower (or someone else) triggers `endAuction()`
9. Execution flow gets hijacked by borrower (lien token owner) inside call to `decreaseYIntercept()`:
    1.  Borrower cancels auction by paying `reservePrice`+`initiatorFee` (this would also go to borrower if `setPayee` is set before the auction as the owner of that Lien token)
    2. Borrower releases NFT to themselves
    3. Borrower now takes another loan of 50 ETH with the same NFT, vault and commitment
10. Once control returns to `AuctionHouse.endAuction`, borrower's new loan's Lien token is deleted
11. `CollateralToken.endAuction` releases borrower's NFT to borrower
12. Malicious borrower steals 50 ETH from vault without any outstanding liens resulting in LP fund loss

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L199
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L210-L224
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L202

## Tool used

Manual Review

## Recommendation

1. Maintain a mapping of active public vaults.
2. Account for malicious lien token owners via lien buyouts.
3. Use reentrancy guards.

# Issue H-16: `VaultImplementation._validateCommitment` may prevent liens that satisfy their terms of `maxPotentialDebt` 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/182 

## Found by 
tives, zzykxx, obront, hansfriese, rvierdiiev, 0xRajeev, Jeiwan

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

## Discussion

**secureum**

Escalate for 2 USDC.

Given the potential impact to different flows/contexts, we still think this is a high-severity impact (not Medium as judged). A majority of the dups (4 of 6) also reported this as a High.

cc @berndartmueller @lucyoa

**sherlock-admin**

 > Escalate for 2 USDC.
> 
> Given the potential impact to different flows/contexts, we still think this is a high-severity impact (not Medium as judged). A majority of the dups (4 of 6) also reported this as a High.
> 
> cc @berndartmueller @lucyoa

You've created a valid escalation for 2 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted.


**sherlock-admin**

> Escalation accepted.
> 

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue H-17: A malicious lien buyer can DoS to cause fund loss/lock 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/181 

## Found by 
0xRajeev

## Summary

A malicious lien buyer can DoS to cause fund loss/lock of other lien holders & borrower collateral.

## Vulnerability Detail

Anyone can call `buyoutLien` to purchase a lien token delivered to an arbitrary receiver. A malicious entity (even the borrower themselves) can purchase a small lien amount with its token delivered to their contract, which can implement `supportsInterface()` with arbitrary code, e.g. revert, that gets executed by the protocol in multiple places giving the attacker control at those points.

## Impact

1. An attacker can revert `endAuction()` to prevent the release of collateral to the winner.
2. An attacker can revert `liquidate()` to prevent the liquidation from even starting.
3. An attacker can revert `_payment()` to prevent any lien payments from succeeding in calls to `makePayment(uint256 collateralId, uint256 paymentAmount)`

Thus, a malicious lien buyer can DoS to cause fund loss/lock of other lien holders & loss/lock of borrower collateral.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L195
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L381
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L613

## Tool used

Manual Review

## Recommendation

1. Maintain a mapping of active public vaults.
2. Account for malicious lien token owners via lien buyouts.
3. Use reentrancy guards.

# Issue H-18: Lien buyout with new terms does not update the slope of public vaults 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/180 

## Found by 
0xRajeev

## Summary

Lien buyout with new terms does not update the slope of public vaults leading to reverts and potential insolvency of vaults.

## Vulnerability Detail

In the `VaultImplementation.buyoutLien` function, a lien is bought out with new terms. Terms of a lien (last, start, rate, duration) will have changed after a buyout. This changes the slope calculation, however, after the buyout, the slope for the public vault is not updated.

## Impact

There are multiple parts of the code affected by this.

1) When liquidating, the vault slope is adjusted by the liens slope (see https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L385-L387). In case of a single lien, the PublicVault.updateVaultAfterLiquidation function can revert if the lien terms have changed previously due to a buyout (https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L532). Hence, liquidations with a public vault involved, will revert.

2) The PublicVault contract calculates the implied value of a vault (ie. totalAssets) with the use of the slope value (see https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L412). As the slope value can be outdated, this leads to undervalue or overvalue of a public vault and thus vault share calculations will be incorrect.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L280-L304
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L150-L153

## Tool used

Manual Review

## Recommendation

Calculate the slope of the lien before the buyout in the `VautImplementation.buyoutLien` function, subtract the calculated slope value from `PublicVault.slope`, update lien terms, recalculate the slope and add the new updated slope value to `PublicVault.slope`.

# Issue H-19: Auction bid that partially pays back an expired lien will revert 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/179 

## Found by 
Prefix, 0xRajeev, sorrynotsorry

## Summary

Auction bids will revert for a bid amount that does not fully pay back an expired lien.

## Vulnerability Detail

The value of `lien.last` is updated to `block.timestamp` in several parts of the code, even for expired liens. Payments made as part of liquidations will set `lien.last` to `block.timestamp` > `lien.start + lien.duration` causing a revert in the flow shown below at step 3 when it is subtracted from `end`:

1. `IPublicVault(lienOwner).afterPayment(lienId);`
2. `slope += LIEN_TOKEN().calculateSlope(lienId);`
3. `return (owedAtEnd - lien.amount).mulDivDown(1, end - lien.last);`

## Impact

Auction bids will revert for a bid amount that does not fully pay back an expired lien. If a lien is only partially paid back, it will reach the `if` branch in `LienToken._payment`, which sets `lien.last` to the current timestamp (which itself is `> lien.start + lien.duration`, hence the liquidation) and then revert.

This affects the economic efficiency of auctions because it DoS's partial bids and therefore results in loss of funds to LPs.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L444
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L625 

## Tool used

Manual Review

## Recommendation

Revisit the logic that updates `lien.last` in the protocol to ensure no reverts in expected flows.

# Issue H-20: Anyone can deposit and mint withdrawal proxy shares to capture distributed yield from borrower interests 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/176 

## Found by 
bin2chen, 0x4141, 0xRajeev, TurnipBoy

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

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

This leads to material loss of funds. Definitely high risk

**sherlock-admin**

 > Escalate for 1 USDC
> 
> This leads to material loss of funds. Definitely high risk

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted

**sherlock-admin**

> Escalation accepted

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue H-21: Epochs can be progressed during ongoing auctions to cause LP fund loss and collateral lockup 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/175 

## Found by 
0xRajeev

## Summary

The `currentEpoch` can be progressed while having an ongoing auction which will completely mess up the liquidation logic to potentially cause LP fund loss and collateral lockup.

## Vulnerability Detail

The `LiquidationAccountant.claim()` function is only callable if `finalAuctionEnd` is set to 0 or the  `block.timestamp` is greater than `finalAuctionEnd` (i.e. auction has ended). Furthermore, `PublicVault.processEpoch` should only be callable if there is *no* ongoing auction if a liquidation accountant is deployed in the current epoch.

`finalAuctionEnd` is set within the `LiquidationAccountant.handleNewLiquidation` function, which is called from the `AstariaRouter.liquidate` function. However, instead of providing a timestamp of the auction end, a value of `2 days + 1 days` is given because `COLLATERAL_TOKEN.auctionWindow()` returns `2 days`.

This does not achieve the intended constraint on checking for the end of the auction because it uses a fixed duration instead of a timestamp.

## Impact

Epochs can be progressed during ongoing auctions to completely mess up the liquidation logic to potentially cause LP fund loss and collateral lockup.
 
## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LiquidationAccountant.sol#L67
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L244-L248
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L409

## Tool used

Manual Review

## Recommendation

Revisit the logic to use an appropriate timestamp instead of a fixed duration.

# Issue H-22: Public vault depositors will receive fewer vault shares until the first payment 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/173 

## Found by 
0xRajeev

## Summary

Public vault total asset calculation is incorrect until the first payment, leading to depositors receiving fewer vault shares than expected.

## Vulnerability Detail

As long as `PublicVault.last` is set to 0, the `PublicVault.totalAssets` function returns the actual ERC-20 token balance (WETH) of the public vault. Due to borrowing, this balance is reduced by the borrowed amount. Therefore, as there is no payment, this leads to depositors receiving fewer vault shares than expected.

## Impact

PoC: https://gist.github.com/berndartmueller/8a71ff76c7eb8207e1f01a154a873b2c

Public vault depositors will receive fewer vault shares until the first payment.
 
## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L407

## Tool used

Manual Review

## Recommendation

Revisit the logic behind updating `last` when `yIntercept` and/or `slope` are updated.

# Issue H-23: Payments and liquidations of multiple liens will revert and can be exploited, causing payer fund loss 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/172 

## Found by 
0xRajeev

## Summary

The `liens` array and its indexes are changed whenever a lien is fully paid back. Therefore, referencing liens by their position in the array is broken because indices change. Payments and liquidations of multiple liens within a loop will revert and can be exploited by sandwiching liens to force a payment to the attacker's lien.

## Vulnerability Detail

Whenever a lien position is fully paid back, the lien is deleted from the `liens` array. This means that the array is shortened and the indices of the remaining liens are changed. This is problematic when paying back liens in a loop because the indices of the remaining liens are changed which will cause revert with an index out-of-bounds error.

One can be exploited to pay for a different (attacker's) lien position because the lien array gets shortened when another payment (by the attacker or someone else) beforehand causes a deleted lien position.


## Impact

Liquidations of multiple liens and payments against a collateral token and all its liens will revert. This will result in the collateral NFT being locked forever.

**Exploit scenario:** A malicious junior debt holder can buy out a senior debt (sandwich the original borrower effectively) to trigger this sequence of actions. For e.g., if the `liens` array is [1, 5, 3] where the malicious junior lien holder of 3 also buys out 1 and front-runs the borrower's intended payment for 5 by repaying 1 and forcing the array to shorten to [5, 3] will make the borrower pay off their lien of 3 instead of borrower's lien of 5. Effectively, a lien holder can "force" a payment to their own lien instead of another one.
 
## Code Snippet
1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L663
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L418
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L296

## Tool used

Manual Review

## Recommendation

Revisit and fix the entire logic that manages positions with the `liens` array addition, traversal and deletion.

# Issue H-24: Loans can exceed the maximum potential debt leading to vault insolvency and possible loss of LP funds 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/169 

## Found by 
0xRajeev

## Summary

Missing to account for the new lien can allow loans on a collateral to exceed maximum potential debt leading to vault insolvency and potential loss of LP funds.

## Vulnerability Detail

The `LienToken.createLien` function tries to prevent loans and the total potential debt from surpassing the defined `params.terms.maxPotentialDebt` limit. When the `getTotalDebtForCollateralToken` function is called, the new lien (which will be created within this transaction) is not yet added to the `liens[collateralId]` array. However, the `getTotalDebtForCollateralToken` function iterates over this very same array and will return the total debt without considering the new lien being added and for which this check is being performed.

## Impact

The strategist's defined max potential debt limit can be exceeded, which changes/increases the risk for LPs, as it imposes a higher debt to the public vault. This could lead to vault insolvency and loss of LP funds.

PoC: https://gist.github.com/berndartmueller/8b0f870962acc4c999822d742e89151b

Example exploit: Given a public vault and a lien with a max potential debt amount of 50 ETH (that's the default `standardLien` in TestHelpers.t.sol)

    1. 100 ETH have been deposited in the public vault by LPs
    2. Bob borrows 50 ETH with his 1st NFT -> success
    3. Bob borrows another 50 ETH with his 2nd NFT -> successful as well even though theirs a limit of 50 ETH 
    4. Bob now has 100 ETH -> The max potential debt limit is exceeded by 50 ETH

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L253-L262

## Tool used

Manual Review

## Recommendation

Check the total debt limit after adding the new lien to the `liens[collateralId]` array.

## Discussion

**androolloyd**

this is working as intended, the value is intended to represent the max potential debt the collateral can potentnially be under at the moment the new terms are originated.

**secureum**

Escalate for 2 USDC.

We still think this is a valid issue with a high-severity impact as described above and demonstrated in the PoC.

cc @berndartmueller @lucyoa 

**sherlock-admin**

 > Escalate for 2 USDC.
> 
> We still think this is a valid issue with a high-severity impact as described above and demonstrated in the PoC.
> 
> cc @berndartmueller @lucyoa 

You've created a valid escalation for 2 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted

Having the possibility of exceeding the maximum potential debt should be mitigated

**sherlock-admin**

> Escalation accepted
> 
> Having the possibility of exceeding the maximum potential debt should be mitigated

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue H-25: Triggering liquidations ahead of expected time leads to loss and lock of funds 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/167 

## Found by 
0xRajeev

## Summary

Triggering liquidations for a collateral after one of its liens has expired but before the auction window (default 2 days) at epoch end leads to loss and lock of funds.

## Vulnerability Detail

Liquidations are allowed to be triggered for a collateral if any of its liens have exceeded their loan duration with outstanding payments. However, the liquidation logic does not execute the decrease of lien count and setting up of liquidation accountant if the time to epoch end is greater than the auction window (default 2 days). The auction proceeds nevertheless.

## Impact

If liquidations are triggered before the auction window at epoch end, they will proceed to auctions without decreasing epoch lien count, without setting up the liquidation accountant for the lien and other related logic. This will, at a minimum, lead to auction proceeds going to lien owners directly instead of via the liquidation accountants (loss of funds) and the epoch unable to proceed to the next on (lock of funds and protocol halt).

## Code Snippet

1.https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L388-L415

## Tool used

Manual Review

## Recommendation

Revisit the liquidation logic and its triggering related to the auction window and epoch end.

# Issue H-26: Lack of access control in PublicVault.sol#transferWithdrawReserve let user call transferWithdrawReserve() multiple times to modify withdrawReserve 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/163 

## Found by 
ctf\_sec, Jeiwan

## Summary

Lack of access control in PublicVault.sol#transferWithdrawReserve let user call transferWithdrawReserve() multiple times to modify withdrawReserve

## Vulnerability Detail

The function PublicVault.sol#transferWithdrawReserve() is meants to transfers funds from the PublicVault to the WithdrawProxy.

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L341-L363

However, this function has no access control, anyone can call it multiple times to modify the withdrawReserve value

## Impact

A malicious actor can keep calling the function transferWthdrawReserve() before the withdrawal proxy is created.

If the underlying proxy has address(0), the transfer is not performed, but the state withdrawReserves is decremented.

Then user can invoke this function to always decrement the withdrawReserve to 0

```solidity
    // prevent transfer of more assets then are available
    if (withdrawReserve <= withdraw) {
      withdraw = withdrawReserve;
      withdrawReserve = 0;
    } else {
      withdrawReserve -= withdraw;
    }
```

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L341-L363

## Tool used

Manual Review

## Recommendation

We recommend the project add requiestAuth modifier to the function 

```solidity
  function transferWithdrawReserve() public {
```

We can also change the implementation by implmenting: if the underlying withdrawProxy is address(0), revert transfer.

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

Invalid. Impossible for withdrawReserve != 0 and there isn't a withdraw proxy, because of the following logic. Anytime there is a liquidation a liquidation accountant is deployed and that ALWAYS deploys a withdraw proxy. If there's no liquidations then there's no withdrawReserves.

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Invalid. Impossible for withdrawReserve != 0 and there isn't a withdraw proxy, because of the following logic. Anytime there is a liquidation a liquidation accountant is deployed and that ALWAYS deploys a withdraw proxy. If there's no liquidations then there's no withdrawReserves.

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation rejected. There's no hard requirement to always have a liquidation accountant..

**sherlock-admin**

> Escalation rejected. There's no hard requirement to always have a liquidation accountant..

This issue's escalations have been rejected!

Watsons who escalated this issue will have their escalation amount deducted from their next payout.



# Issue H-27: Bidder can cheat auction by placing bid much higher than reserve price when there are still open liens against a token 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/107 

## Found by 
TurnipBoy

## Summary

When a token still has open liens against it only the value of the liens will be paid by the bidder but their current bid will be set to the full value of the bid. This can be abused in one of two ways. The bidder could place a massive bid like 500 ETH that will never be outbid or they could place a bid they know will outbid and profit the difference when they're sent a refund.

## Vulnerability Detail

    uint256[] memory liens = LIEN_TOKEN.getLiens(tokenId);
    uint256 totalLienAmount = 0;
    if (liens.length > 0) {
      for (uint256 i = 0; i < liens.length; ++i) {
        uint256 payment;
        uint256 lienId = liens[i];

        ILienToken.Lien memory lien = LIEN_TOKEN.getLien(lienId);

        if (transferAmount >= lien.amount) {
          payment = lien.amount;
          transferAmount -= payment;
        } else {
          payment = transferAmount;
          transferAmount = 0;
        }
        if (payment > 0) {
          LIEN_TOKEN.makePayment(tokenId, payment, lien.position, payer);
        }
      }
    } else {
      //@audit-issue logic skipped if liens.length > 0
      TRANSFER_PROXY.tokenTransferFrom(
        weth,
        payer,
        COLLATERAL_TOKEN.ownerOf(tokenId),
        transferAmount
      );
    }

We can examine the payment logic inside `_handleIncomingPayment` and see that if there are still open liens against then only the amount of WETH to pay back the liens will be taken from the payer, since the else portion of the logic will be skipped.

    uint256 vaultPayment = (amount - currentBid);

    if (firstBidTime == 0) {
      auctions[tokenId].firstBidTime = block.timestamp.safeCastTo64();
    } else if (lastBidder != address(0)) {
      uint256 lastBidderRefund = amount - vaultPayment;
      _handleOutGoingPayment(lastBidder, lastBidderRefund);
    }
    _handleIncomingPayment(tokenId, vaultPayment, address(msg.sender));

    auctions[tokenId].currentBid = amount;
    auctions[tokenId].bidder = address(msg.sender);

In `createBid`, `auctions[tokenId].currentBid` is set to `amount` after the last bidder is refunded and the excess is paid against liens. We can walk through an example to illustrate this:

Assume a token with a single lien of amount 10 WETH and an auction is opened for that token. Now a user places a bid for 20 WETH. They are the first bidder so `lastBidder = address(0)` and `currentBid = 0`. `_handleIncomingPayment` will be called with a value of 20 WETH since there is no lastBidder to refund. Inside `_handleIncomingPayment` the lien information is read showing 1 lien against the token. Since `transferAmount >= lien.amount`, `payment = lien.amount`. A payment will be made by the bidder against the lien for 10 WETH. After the payment `_handleIncomingPayment` will return only having taken 10 WETH from the bidder. In the next line currentBid is set to 20 WETH but the bidder has only paid 10 WETH. Now if they are outbid, the new bidder will have to refund then 20 WETH even though they initially only paid 10 WETH.

## Impact

Bidder can steal funds due to `_handleIncomingPayment` not taking enough WETH

## Code Snippet

https://github.com/AstariaXYZ/astaria-gpl/blob/64acee1122a71b23eef037f69cef4c0c087241be/src/AuctionHouse.sol#L250-L304

## Tool used

Manual Review

## Recommendation

In `_handleIncomingPayment`, all residual transfer amount should be sent to `COLLATERAL_TOKEN.ownerOf(tokenId)`.

# Issue H-28: Possible to fully block PublicVault.processEpoch function. No one will be able to receive their funds 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/88 

## Found by 
TurnipBoy, rvierdiiev

## Summary
Possible to fully block `PublicVault.processEpoch` function. No one will be able to receive their funds
## Vulnerability Detail
When liquidity providers want to redeem their share from `PublicVault` they call `redeemFutureEpoch` function which will create new `WithdrawProxy` for the epoch(if not created already) and then mint shares for redeemer in `WithdrawProxy`. PublicVault [transfer](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L190) user's shares to himself.

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L178-L212
```solidity
  function redeemFutureEpoch(
    uint256 shares,
    address receiver,
    address owner,
    uint64 epoch
  ) public virtual returns (uint256 assets) {
    // check to ensure that the requested epoch is not the current epoch or in the past
    require(epoch >= currentEpoch, "Exit epoch too low");


    require(msg.sender == owner, "Only the owner can redeem");
    // check for rounding error since we round down in previewRedeem.


    ERC20(address(this)).safeTransferFrom(owner, address(this), shares);


    // Deploy WithdrawProxy if no WithdrawProxy exists for the specified epoch
    _deployWithdrawProxyIfNotDeployed(epoch);


    emit Withdraw(msg.sender, receiver, owner, assets, shares);


    // WithdrawProxy shares are minted 1:1 with PublicVault shares
    WithdrawProxy(withdrawProxies[epoch]).mint(receiver, shares); // was withdrawProxies[withdrawEpoch]
  }
```

This function mints `WithdrawProxy` shares 1:1 to redeemed `PublicVault` shares.
Then later after call of `processEpoch` and `transferWithdrawReserve` the funds will be sent to the WithdrawProxy and users can now redeem their shares from it.

Function `processEpoch` decides how many funds should be sent to the `WithdrawProxy`.

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L268-L289
```solidity
    if (withdrawProxies[currentEpoch] != address(0)) {
      uint256 proxySupply = WithdrawProxy(withdrawProxies[currentEpoch])
        .totalSupply();


      liquidationWithdrawRatio = proxySupply.mulDivDown(1e18, totalSupply());


      if (liquidationAccountants[currentEpoch] != address(0)) {
        LiquidationAccountant(liquidationAccountants[currentEpoch])
          .setWithdrawRatio(liquidationWithdrawRatio);
      }


      uint256 withdrawAssets = convertToAssets(proxySupply);
      // compute the withdrawReserve
      uint256 withdrawLiquidations = liquidationsExpectedAtBoundary[
        currentEpoch
      ].mulDivDown(liquidationWithdrawRatio, 1e18);
      withdrawReserve = withdrawAssets - withdrawLiquidations;
      // burn the tokens of the LPs withdrawing
      _burn(address(this), proxySupply);


      _decreaseYIntercept(withdrawAssets);
    }
```

This is how it is decided how much money should be sent to WithdrawProxy.
Firstly, we look at totalSupply of WithdrawProxy. 
`uint256 proxySupply = WithdrawProxy(withdrawProxies[currentEpoch]).totalSupply();`.

And then we convert them to assets amount.
`uint256 withdrawAssets = convertToAssets(proxySupply);`

In the end function burns `proxySupply` amount of shares controlled by PublicVault.
` _burn(address(this), proxySupply);`

Then this amount is allowed to be sent(if no auctions currently, but this is not important right now).

This all allows to attacker to make `WithdrawProxy.deposit` to mint new shares for him and increase totalSupply of WithdrawProxy, so `proxySupply` becomes more then was sent to `PublicVault`.

This is attack scenario.

1.PublicVault is created and funded with 50 ethers.
2.Someone calls `redeemFutureEpoch` function to create new WithdrawProxy for next epoch.
3.Attacker sends 1 wei to WithdrawProxy to make totalAssets be > 0. Attacker deposit to WithdrawProxy 1 wei. Now WithdrawProxy.totalSupply > PublicVault.balanceOf(PublicVault).
4.Someone call `processEpoch` and it reverts on burning.

As result, nothing will be send to WithdrawProxy where shares were minted for users. The just lost money.

Also this attack can be improved to drain users funds to attacker. 
Attacker should be liquidity provider. And he can initiate next redeem for next epoch, then deposit to new WithdrawProxy enough amount to get new shares. And call `processEpoch` which will send to the vault amount, that was not sent to previous attacked WithdrawProxy, as well. So attacker will take those funds.
## Impact
Funds of PublicVault depositors are stolen.
## Code Snippet
This is simple test that shows how external actor can corrupt WithdrawProxy.

```solidity
function testWithdrawProxyDdos() public {
    Dummy721 nft = new Dummy721();
    address tokenContract = address(nft);
    uint256 tokenId = uint256(1);

    address publicVault = _createPublicVault({
      strategist: strategistOne,
      delegate: strategistTwo,
      epochLength: 14 days
    });

    _lendToVault(
      Lender({addr: address(1), amountToLend: 50 ether}),
      publicVault
    );

    uint256 collateralId = tokenContract.computeId(tokenId);

    uint256 vaultTokenBalance = IERC20(publicVault).balanceOf(address(1));
    console.log("balance: ", vaultTokenBalance);

    // _signalWithdrawAtFutureEpoch(address(1), publicVault, uint64(1));
    _signalWithdraw(address(1), publicVault);

    address withdrawProxy = PublicVault(publicVault).withdrawProxies(
      PublicVault(publicVault).getCurrentEpoch()
    );

    vm.deal(address(2), 2);
    vm.startPrank(address(2));
      WETH9.deposit{value: 2}();
      //this we need to make share calculation not fail with divide by 0
      WETH9.transferFrom(address(2), withdrawProxy, 1);
      WETH9.approve(withdrawProxy, 1);
      
      //deposit 1 wei to make WithdrawProxy.totalSupply > PublicVault.balanceOf(address(PublicVault))
      //so processEpoch can't burn more shares
      WithdrawProxy(withdrawProxy).deposit(1, address(2));
    vm.stopPrank();

    assertEq(vaultTokenBalance, IERC20(withdrawProxy).balanceOf(address(1)));

    vm.warp(block.timestamp + 15 days);

    //processEpoch fails, because it can't burn more amount of shares, that was sent to PublicVault
    vm.expectRevert();
    PublicVault(publicVault).processEpoch();
  }
```
## Tool used

Manual Review

## Recommendation
Make function WithdrawProxy.deposit not callable.

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

Causes total loss of funds for LPs when this happens. High risk

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Causes total loss of funds for LPs when this happens. High risk

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted

**sherlock-admin**

> Escalation accepted

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue H-29: nlrType type is not signed by strategist, which could allow fraudulent behavior as new types are added 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/72 

## Found by 
obront

## Summary

The strategist signs the merkle root, their nonce, and the deadline of all strategies to ensure that new borrowers meet their criteria. However, the lien type (`nlrType`) is not signed. Currently, the structs for the different types are unique, so there is no ability to borrow one type as another, but if struct schemas of different types overlap in the future, this will open the door for exploits.

## Vulnerability Detail

When a new lien is requested, the borrower submits a Lien Request, which is filled with the parameters at which they would like to borrow. This is kept honest and aligned with the lenders intent because the merkle root, strategist nonce, and deadline are all signed by the strategist.

Because the merkle root is signed, the borrower must submit lien parameters (`nlrDetails`) that align with one of the strategies that the strategist has chosen to allow (represented as leaves in the merkle tree). The schemas of these signed structs differ depending on the validator being used, which is defined in the `nlrType` parameter.

Currently, each of the validators has a unique schema for their struct. However, if there is an overlap in the schema of the Details struct of multiple validators, where the different parameters represent different values, it opens the door to having a fraudulent lien accepted.

Here's an example of how this might work:
- Type A has a Details struct with the shape { uint8 version, bool requirementX, IAstariaRouter.LienDetails lien }. 
- The lender includes in their merkle tree the following { version: 1, requirementX: true, lien: { ... maxAmount: 1 ether ... }
- Type B has a Details struct with the shape { uint8 version, bool requirementY, IAstariaRouter.LienDetails lien }
- The lender includes in their merkle tree the following strategy: { version: 1, requirementY: true, lien { ... maxAmount: 1 ether ... }
- The lender signs a merkle root including both of these strategies
- A borrower who meets requirementX but not requirementY could submit `lienDetails` with `nlrType = Type A` and send the validation to the wrong strategy validator, thus bypassing the expected checks

## Impact

As more strategy types are added, conflicts in struct schemas could open the door to fraudulent behavior.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L222-L232

Current Details struct schemas:

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/strategies/CollectionValidator.sol#L19-L24

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/strategies/UNI_V3Validator.sol#L21-L31

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/strategies/UniqueValidator.sol#L19-L25

## Tool used

Manual Review

## Recommendation

Include the `nlrType` in the data signed by the strategist. The easiest way to do this would be to pack it in with each leaf when assembling the leaves that will create the merkle tree:

```solidity
function assembleLeaf(ICollectionValidator.Details memory details, address nlrType)
  public
  pure
  returns (bytes memory)
{
  return abi.encode(details, nlrType);
}

function validateAndParse... {
  ...
  leaf = keccak256(assembleLeaf(cd, params.nlrType));
}
```

# Issue H-30: Auctions can end in epoch after intended, underpaying withdrawers 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/51 

## Found by 
obront

## Summary

When liens are liquidated, the router checks if the auction will complete in a future epoch and, if it does, sets up a liquidation accountant and other logistics to account for it. However, the check for auction completion does not take into account extended auctions, which can therefore end in an unexpected epoch and cause accounting issues, losing user funds.

## Vulnerability Detail

The liquidate() function performs the following check to determine if it should set up the liquidation to be paid out in a future epoch:

```solidity
if (PublicVault(owner).timeToEpochEnd() <= COLLATERAL_TOKEN.auctionWindow())
```
This function assumes that the auction will only end in a future epoch if the `auctionWindow` (typically set to 2 days) pushes us into the next epoch.

However, auctions can last up to an additional 1 day if bids are made within the final 15 minutes. In these cases, auctions are extended repeatedly, up to a maximum of 1 day.

```solidity
if (firstBidTime + duration - block.timestamp < timeBuffer) {
  uint64 newDuration = uint256(
    duration + (block.timestamp + timeBuffer - firstBidTime)
  ).safeCastTo64();
  if (newDuration <= auctions[tokenId].maxDuration) {
    auctions[tokenId].duration = newDuration;
  } else {
    auctions[tokenId].duration =
      auctions[tokenId].maxDuration -
      firstBidTime;
  }
  extended = true;
}
```
The result is that there are auctions for which accounting is set up for them to end in the current epoch, but will actual end in the next epoch. 

## Impact

Users who withdrew their funds in the current epoch, who are entitled to a share of the auction's proceeds, will not be paid out fairly.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L388-L415

https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L127-L146

## Tool used

Manual Review

## Recommendation

Change the check to take the possibility of extension into account:

```solidity
if (PublicVault(owner).timeToEpochEnd() <= COLLATERAL_TOKEN.auctionWindow() + 1 days)
```

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

The specific issue address in this submission is incorrect. It passes COLLATERAL_TOKEN.auctionWindow() + 1 days (max possible duration of an auction) into handleNewLiquidation which makes this a non issue. 

Relevant Lines:
https://github.com/sherlock-audit/2022-10-astaria/blob/7d12a5516b7c74099e1ce6fb4ec87c102aec2786/src/AstariaRouter.sol#L407-L410

**sherlock-admin**

 > Escalate for 1 USDC
> 
> The specific issue address in this submission is incorrect. It passes COLLATERAL_TOKEN.auctionWindow() + 1 days (max possible duration of an auction) into handleNewLiquidation which makes this a non issue. 
> 
> Relevant Lines:
> https://github.com/sherlock-audit/2022-10-astaria/blob/7d12a5516b7c74099e1ce6fb4ec87c102aec2786/src/AstariaRouter.sol#L407-L410

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation rejected. 

The `if` in https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L388-L391 is missing the auction window extension of 1 day.  This leads to auctions with extended durations overlapping the current epoch and not having liquidation accountants in place

**sherlock-admin**

> Escalation rejected. 
> 
> The `if` in https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L388-L391 is missing the auction window extension of 1 day.  This leads to auctions with extended durations overlapping the current epoch and not having liquidation accountants in place

This issue's escalations have been rejected!

Watsons who escalated this issue will have their escalation amount deducted from their next payout.



# Issue H-31: Claiming liquidationAccountant will reduce vault y-intercept by more than the correct amount 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/48 

## Found by 
obront

## Summary

When `claim()` is called on the Liquidation Accountant, it decreases the y-intercept based on the balance of the contract after funds have been distributed, rather than before. The result is that the y-intercept will be decreased more than it should be, siphoning funds from all users.

## Vulnerability Detail

When `LiquidationAccountant.sol:claim()` is called, it uses its `withdrawRatio` to send some portion of its earnings to the `WITHDRAW_PROXY` and the rest to the vault.

After performing these transfers, it updates the vault's y-intercept, decreasing it by the gap between the expected return from the auction, and the reality of how much was sent back to the vault:

```solidity
PublicVault(VAULT()).decreaseYIntercept(
  (expected - ERC20(underlying()).balanceOf(address(this))).mulDivDown(
    1e18 - withdrawRatio,
    1e18
  )
);
```
This rebalancing uses the balance of the `liquidationAccountant` to perform its calculation, but it is done after the balance has already been distributed, so it will always be 0.

Looking at an example:
- `expected = 1 ether` (meaning the y-intercept is currently based on this value)
- `withdrawRatio = 0` (meaning all funds will go back to the vault)
- The auction sells for exactly 1 ether
- 1 ether is therefore sent directly to the vault
- In this case, the y-intercept should not be updated, as the outcome was equal to the expected outcome
- However, because the calculation above happens after the funds are distributed, the decrease equals `(expected - 0) * 1e18 / 1e18`, which equals `expected`

That decrease should not happen, and causing problems for the protocol's accounting. For example, when `withdraw()` is called, it uses the y-intercept in its calculation of the `totalAssets()` held by the vault, creating artificially low asset values for a given number of shares.

## Impact

Every time the liquidation accountant is used, the vault's math will be thrown off and user shares will be falsely diluted.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LiquidationAccountant.sol#L62-L97

## Tool used

Manual Review

## Recommendation

The amount of assets sent to the vault has already been calculated, as we've already sent it. Therefore, rather than the full existing formula, we can simply call:

```solidity
PublicVault(VAULT()).decreaseYIntercept(expected - balance)
```

Alternatively, we can move the current code above the block of code that transfers funds out (L73).

# Issue H-32: Incorrect fees will be charged 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/36 

## Found by 
csanuragjain

## Summary
If user has provided transferAmount which is greater than all lien.amount combined then initiatorPayment will be incorrect since it is charged on full amount when only partial was used as shown in poc

## Vulnerability Detail
1. Observe the _handleIncomingPayment function
2. Lets say transferAmount was 1000
3. initiatorPayment is calculated on this full transferAmount

```python
uint256 initiatorPayment = transferAmount.mulDivDown(
      auction.initiatorFee,
      100
    ); 
```

4. Now all lien are iterated and lien.amount is kept on deducting from transferAmount until all lien are navigated

```python
if (transferAmount >= lien.amount) {
          payment = lien.amount;
          transferAmount -= payment;
        } else {
          payment = transferAmount;
          transferAmount = 0;
        }

        if (payment > 0) {
          LIEN_TOKEN.makePayment(tokenId, payment, lien.position, payer);
        }
      }
```

5. Lets say after loop completes the transferAmount is still left as 100 
6. This means only 400 transferAmount was used but fees was deducted on full amount 500

## Impact
Excess initiator fees will be deducted which was not required

## Code Snippet
https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L276

## Tool used
Manual Review

## Recommendation
Calculate the exact amount of transfer amount required for the transaction and calculate the initiator fee based on this amount

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

The fee is not the problem, so this report is invalid. The issue is that the payment isn't working as intended and that sometimes it doesn't take as much as it should. See #107.

**sherlock-admin**

 > Escalate for 1 USDC
> 
> The fee is not the problem, so this report is invalid. The issue is that the payment isn't working as intended and that sometimes it doesn't take as much as it should. See #107.

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation rejected.

This is a valid finding as fees are too high - Fees should be calculated based on the actual amount used as the repayment

**sherlock-admin**

> Escalation rejected.
> 
> This is a valid finding as fees are too high - Fees should be calculated based on the actual amount used as the repayment

This issue's escalations have been rejected!

Watsons who escalated this issue will have their escalation amount deducted from their next payout.



# Issue H-33: isValidRefinance checks both conditions instead of one, leading to rejection of valid refinances 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/22 

## Found by 
obront

## Summary

`isValidRefinance()` is intended to check whether either (a) the loan interest rate decreased sufficiently or (b) the loan duration increased sufficiently. Instead, it requires both of these to be true, leading to the rejection of valid refinances.

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

The currently implementation of the code requires both of these conditions to be met:

```solidity
return (
    newLien.rate >= minNewRate &&
    ((block.timestamp + newLien.duration - lien.start - lien.duration) >= minDurationIncrease)
);
```

## Impact

Valid refinances that meet one of the two criteria will be rejected.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L488-L490

## Tool used

Manual Review

## Recommendation

Change the AND in the return statement to an OR:

```solidity
return (
    newLien.rate >= minNewRate ||
    ((block.timestamp + newLien.duration - lien.start - lien.duration) >= minDurationIncrease)
);
```

## Discussion

**SantiagoGregory**

Independently fixed during our own review so there's no PR specifically for this, but this is now updated to an or.

**IAmTurnipBoy**

Escalate for 1 USDC

Should be medium because there are no funds at risk

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Should be medium because there are no funds at risk

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation rejected.

Not a major loss of funds but definitely a severe flaw that will hurt the protocol.

**sherlock-admin**

> Escalation rejected.
> 
> Not a major loss of funds but definitely a severe flaw that will hurt the protocol.

This issue's escalations have been rejected!

Watsons who escalated this issue will have their escalation amount deducted from their next payout.



# Issue H-34: isValidRefinance will approve invalid refinances and reject valid refinances due to buggy math 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/21 

## Found by 
0xRajeev, obront, hansfriese

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

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

Should be medium because no funds at risk

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Should be medium because no funds at risk

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation rejected.

Not a major loss of funds but definitely a severe flaw that will hurt the protocol.

**sherlock-admin**

> Escalation rejected.
> 
> Not a major loss of funds but definitely a severe flaw that will hurt the protocol.

This issue's escalations have been rejected!

Watsons who escalated this issue will have their escalation amount deducted from their next payout.



# Issue M-1: new loans "max duration" is not restricted 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/264 

## Found by 
bin2chen

## Summary
document :
"
Epochs
[PublicVaults](https://docs.astaria.xyz/docs/smart-contracts/PublicVault) operate around a time-based epoch system. An epoch length is defined by the strategist that deploys the [PublicVault](https://docs.astaria.xyz/docs/smart-contracts/PublicVault). The duration of new loans is restricted to not exceed the end of the next epoch. For example, if a [PublicVault](https://docs.astaria.xyz/docs/smart-contracts/PublicVault) is 15 days into a 30-day epoch, new loans must not be longer than 45 days.
"
but more than 2 epoch's duration can be added

## Vulnerability Detail
the max duration is not detected. add success when > next epoch

#AstariaTest#testBasicPublicVaultLoan

```solidity
  function testBasicPublicVaultLoan() public {

  IAstariaRouter.LienDetails memory standardLien2 =
    IAstariaRouter.LienDetails({
      maxAmount: 50 ether,
      rate: (uint256(1e16) * 150) / (365 days),
      duration: 50 days,  /****** more then 14 * 2 *******/
      maxPotentialDebt: 50 ether
    });    

    _commitToLien({
      vault: publicVault,
      strategist: strategistOne,
      strategistPK: strategistOnePK,
      tokenContract: tokenContract,
      tokenId: tokenId,
      lienDetails: standardLien2, /**** use standardLien2 ****/
      amount: 10 ether,
      isFirstLien: true
    });
  }
```

## Impact
Too long duration

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L209

## Tool used

Manual Review

## Recommendation
PublicVault#_afterCommitToLien
```solidity
  function _afterCommitToLien(uint256 lienId, uint256 amount)
    internal
    virtual
    override
  {
    // increment slope for the new lien
    unchecked {
      slope += LIEN_TOKEN().calculateSlope(lienId);
    }

    ILienToken.Lien memory lien = LIEN_TOKEN().getLien(lienId);

    uint256 epoch = Math.ceilDiv(
      lien.start + lien.duration - START(),
      EPOCH_LENGTH()
    ) - 1;

+   require(epoch <= currentEpoch + 1,"epoch max <= currentEpoch + 1");

    liensOpenForEpoch[epoch]++;
    emit LienOpen(lienId, epoch);
  }


```

# Issue M-2: If an auction has no bidder, the NFT ownership should go back to the loan lenders 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/258 

## Found by 
\_\_141345\_\_

## Summary

The lenders in principal have the claim for the loan collateral, but current rule will let the liquidation caller get the collateral for free. Effectively take advantage from the vault LP, which is not fair.


## Vulnerability Detail

After the `endAuction()`, the collateral will be released to the initiator. Essentially, the initiator gets the NFT for free. But the lenders of the loan take the loss.

However, the lenders should have the claim to the collateral, since originally the funds are provided by the lenders. If the collateral at the end is owned by whoever calls the liquidation function, it is not fair for the lenders. And will discourage future users to use the protocol.


## Impact

- Lenders could suffer fund loss in some cases.
- The unfair mechanism will discourage future users.


## Code Snippet

If there is no bidder, the winner will be assigned to the auction initiator. And the debts will all be wrote off.
https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L178-L204

After the `endAuction()`, the collateral will be released to the initiator. 
https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/CollateralToken.sol#L341-L346


## Tool used

Manual Review

## Recommendation

If there is no bidder for the auction, allow the NFT to get auctioned for another chance.



## Discussion

**androolloyd**

working as intended

**141345**

Escalate for 3 USDC

A borrower uses the NFT as collateral, the lender will get the collateral if the borrower defaults, that's how lending works normally. However, according to the current rule, anyone starts the liquidation process could potentially get the collateral, if no bidder bid on the auction. And the liquidator initiator already gets compensated by the initiator fee. 

Current rule allows for a situation that 3rd user could gain the ownership of the NFT by calling `liquidate()`. But in common practice, it is the lender should claim the ownership of the collateral.

One step further, if by any chance, the initiator could start some DoS attack and make the protocol inoperable, this rule may become part of the attack, to get the collateral for free.

Although it is a corner case, I believe this is a business logic issue.


**sherlock-admin**

 > Escalate for 3 USDC
> 
> A borrower uses the NFT as collateral, the lender will get the collateral if the borrower defaults, that's how lending works normally. However, according to the current rule, anyone starts the liquidation process could potentially get the collateral, if no bidder bid on the auction. And the liquidator initiator already gets compensated by the initiator fee. 
> 
> Current rule allows for a situation that 3rd user could gain the ownership of the NFT by calling `liquidate()`. But in common practice, it is the lender should claim the ownership of the collateral.
> 
> One step further, if by any chance, the initiator could start some DoS attack and make the protocol inoperable, this rule may become part of the attack, to get the collateral for free.
> 
> Although it is a corner case, I believe this is a business logic issue.
> 

You've created a valid escalation for 3 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted.

Will be rewarded a medium as it requires the auction to end with 0 bids

**sherlock-admin**

> Escalation accepted.
> 
> Will be rewarded a medium as it requires the auction to end with 0 bids

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue M-3: _makePayment is logically inconsistent with how lien stack is managed causing payments to multiple liens to fail 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/232 

## Found by 
TurnipBoy

## Summary

`_makePayment(uint256, uint256)` looping logic is inconsistent with how `_deleteLienPosition` manages the lien stack. `_makePayment` loops from 0 to `openLiens.length` but `_deleteLienPosition` (called when a lien is fully paid off) actively compresses the lien stack. When a payment pays off multiple liens the compressing effect causes an array OOB error towards the end of the loop.

## Vulnerability Detail

    function _makePayment(uint256 collateralId, uint256 totalCapitalAvailable)
      internal
    {
      uint256[] memory openLiens = liens[collateralId];
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
    }

`LienToken.sol#_makePayment(uint256, uint256)` loops from 0 to `openLiens.Length`. This loop attempts to make a payment to each lien calling `_payment` with the current index of the loop.

    function _deleteLienPosition(uint256 collateralId, uint256 position) public {
      uint256[] storage stack = liens[collateralId];
      require(position < stack.length, "index out of bounds");

      emit RemoveLien(
        stack[position],
        lienData[stack[position]].collateralId,
        lienData[stack[position]].position
      );
      for (uint256 i = position; i < stack.length - 1; i++) {
        stack[i] = stack[i + 1];
      }
      stack.pop();
    }

`LienToken.sol#_deleteLienPosition` is called on liens when they are fully paid off. The most interesting portion of the function is how the lien is removed from the stack. We can see that all liens above the lien in question are slid down the stack and the top is popped. This has the effect of reducing the total length of the array. This is where the logical inconsistency is. If the first lien is paid off, it will be removed and the formerly second lien will now occupy it's index. So then when `_payment` is called in the next loop with the next index it won't reference the second lien since the second lien is now in the first lien index.

Assuming there are 2 liens on some collateral. `liens[0].amount = 100` and `liens[1].amount = 50`. A user wants to pay off their entire lien balance so they call  `_makePayment(uint256, uint256)` with an amount of 150. On the first loop it calls `_payment` with an index of 0. This pays off `liens[0]`. `_deleteLienPosition` is called with index of 0 removing `liens[0]`. Because of the sliding logic in `_deleteLienPosition` `lien[1]` has now slid into the `lien[0]` position. On the second loop it calls `_payment` with an index of 1. When it tries to grab the data for the lien at that index it will revert due to OOB error because the array no long contains an index of 1.

## Impact

Large payment are impossible and user must manually pay off each liens separately 

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L410-L424

## Tool used

Manual Review

## Recommendation

Payment logic inside of `AuctionHouse.sol` works. `_makePayment` should be changed to mimic that logic.



## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

Not a dupe of #190, issue is with the lien array is managed as the payments are made. Fixing #190 wouldn't fix this.

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Not a dupe of #190, issue is with the lien array is managed as the payments are made. Fixing #190 wouldn't fix this.

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepeted



# Issue M-4: Outstanding debt is not guaranteed to be covered by auctions 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/203 

## Found by 
0xRajeev

## Summary

The best-effort one-time English auction for borrower collateral is not economically efficient to drive auction bids towards reaching the total outstanding debt, which leads to loss of LP funds.

## Vulnerability Detail

When any lien against a borrower collateral is not paid within the lien duration, the underlying collateral is put up for auction where bids can come in at any price. The borrower is allowed to cancel the auction if the current bid is lower than the reserve price which is set to the total outstanding debt. The reserve price is not enforced anywhere else. If there are no bids, the liquidator will receive the collateral.

## Impact

This auction design of a best-effort one-time English auction is not economically efficient to drive auction bids towards reaching the total outstanding debt which effectively leads to loss of LP funds on unpaid liens.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L210-L217
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L178-L182

## Tool used

Manual Review

## Recommendation

Consider alternative auction design mechanisms e.g. a Dutch auction where the auction starts at the reserve price to provide a higher payment possibility to the LPs.

## Discussion

**SantiagoGregory**

We're switching to a Dutch auction through Seaport.

**Evert0x**

Downgrading to info as it's a protocol design choice.

**secureum**

Escalate for 2 USDC.

This finding is based on current protocol design and implementation (i.e. there was no documentation suggesting their future switch to Dutch auction). Based on the protocol team's response above, they effectively confirm the current design choice (_and_ implementation) to be a serious enough issue that they are changing the protocol design to what is recommended by this finding. Just because it is a design issue does not deem this to be downgraded to informational — design drives implementation and is harder to change. Moving to a Dutch auction, as recommended, will affect significant parts of protocol implementation.

Therefore, we still think this is of Medium severity impact, if not higher.

cc @berndartmueller @lucyoa

**sherlock-admin**

 > Escalate for 2 USDC.
> 
> This finding is based on current protocol design and implementation (i.e. there was no documentation suggesting their future switch to Dutch auction). Based on the protocol team's response above, they effectively confirm the current design choice (_and_ implementation) to be a serious enough issue that they are changing the protocol design to what is recommended by this finding. Just because it is a design issue does not deem this to be downgraded to informational — design drives implementation and is harder to change. Moving to a Dutch auction, as recommended, will affect significant parts of protocol implementation.
> 
> Therefore, we still think this is of Medium severity impact, if not higher.
> 
> cc @berndartmueller @lucyoa

You've created a valid escalation for 2 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted based on comment from Watson

**sherlock-admin**

> Escalation accepted based on comment from Watson

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue M-5: Extension logic incorrectly extends the auction by an additional amount of existing duration 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/201 

## Found by 
minhquanym, bin2chen, yixxas, Prefix, 0xRajeev

## Summary

Incorrect auction extension logic extends the auction by an additional amount of the previous duration instead of extending it by 15 minutes.

## Vulnerability Detail

The calculation of `newDuration` incorrectly adds `duration` in the auction extension logic. This causes the new duration to be extended by an additional amount of the existing duration, instead of an additional 15 minutes (`timeBuffer`), when a bid is created in the last 15 mins of the existing auction duration.

## Impact

Delayed payout of funds/collateral upon auction completion only after the newly extended duration.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L135-L137

## Tool used

Manual Review

## Recommendation

Change the calculation to `uint64 newDuration = uint256(block.timestamp + timeBuffer - firstBidTime).safeCastTo64();`

# Issue M-6: `AstariaRouter.commitToLiens` will revert if the protocol fee is enabled 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/195 

## Found by 
0xRajeev

## Summary

The function `commitToLiens()` will revert in `getProtocolFee()`, which prevents borrowers from depositing collateral and requesting loans in the protocol.

## Vulnerability Detail

If the protocol fee is enabled by setting `feeTo` to a non-zero address, then `getProtocolFee()` will revert because of division-by-zero given that `protocolFeeDenominator` is `0` without any initialization and no setter (in `file()`) for setting it.
 
## Impact

The function `commitToLiens()` will revert if the protocol fee is enabled thus preventing borrowers from depositing collateral and requesting loans in the protocol thereby failing to bootstrap its core NFT lending functionality.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L66-L67
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L441-L443
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L337-L340
4. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L331
5. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L242-L250

## Tool used

Manual Review

## Recommendation

Initialize protocol fee numerator and denominator in `AstariaRouter` and add their setters to `file()`.

## Discussion

**secureum**

Escalate for 2 USDC.

We do not think this is a duplicate issue of #204. While both are about `commitToLiens()` reverting, the triggering locations, conditions and therefore the recommendations are entirely different. This issue is specific to non-initialization of protocol fee variables as described above.

cc @berndartmueller @lucyoa

**sherlock-admin**

 > Escalate for 2 USDC.
> 
> We do not think this is a duplicate issue of #204. While both are about `commitToLiens()` reverting, the triggering locations, conditions and therefore the recommendations are entirely different. This issue is specific to non-initialization of protocol fee variables as described above.
> 
> cc @berndartmueller @lucyoa

You've created a valid escalation for 2 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted

**sherlock-admin**

> Escalation accepted

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue M-7: `LienToken.createLien` may prevent liens that satisfy their terms of `maxPotentialDebt` 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/192 

## Found by 
0xRajeev, hansfriese

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

# Issue M-8: Incorrect `LienToken.changeInSlope` calculation can lead to vault insolvency 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/189 

## Found by 
0xRajeev, hansfriese

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

# Issue M-9: Auctions run for less time than intended 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/186 

## Found by 
chainNue, obront, hansfriese, peanuts, 0xRajeev, csanuragjain

## Summary

Auctions run for less time than intended causing them to be not economically efficient for the lenders, thus causing a suboptimal credit of liquidation funds to them.

## Vulnerability Detail

From the documentation/comments, we can infer that `firstBidTime` is supposed to be `// The time of the first bid` and duration is supposed to be `// The length of time to run the auction for, after the first bid was made.`

However, when an auction is created in `createAuction()`, the auction's `firstBidTime` is incorrectly initialized as `block.timestamp.safeCastTo64()` instead of `0`. This is premature initialization because the auction was only created here and no bid has been made yet via `createBid()`. The code in `createBid()` which check for `firstBidTime == 0 `is more evidence that the initialization in `createAuction()` is incorrect.

## Impact

This causes auctions to run for less time than intended if the first bid comes at a much later time after the auction was created. A shorter auction time could potentially allow fewer bids and cause it to be not economically efficient for the lenders, thus causing a suboptimal credit of liquidation funds to them.

## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/interfaces/IAuctionHouse.sol#L12-L13
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L80
3. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L166-L176

## Tool used

Manual Review

## Recommendation

`createAuction()` should initialize `firstBidTime = 0`.

## Discussion

**androolloyd**

will fix but its a convention issue we want auctions to start immediately not on first bid



# Issue M-10: Minting public vault shares while the protocol is paused can lead to LP fund loss 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/174 

## Found by 
TurnipBoy, 0xRajeev

## Summary

Assets can be deposited into public vaults by LPs with `PublicVault.mint` function to bypass a possible paused protocol.

## Vulnerability Detail

The PublicVault contract prevents calls to the `PublicVault.deposit` function while the protocol is paused by using the `whenNotPaused` modifier.

The `PublicVault` contract extends the `ERC4626Cloned` contract, which has two functions to deposit assets into the vault: the `deposit` function and the `mint` function. The latter function, however, is not overwritten in the PublicVault contract and therefore lacks the appropriate `whenNotPaused` modifier. 

## Impact

LPs can deposit assets into public vaults with the `PublicVault.mint` function to bypass a possible paused protocol. This can lead to LP fund loss depending on the reason for the protocol pause and the incident response.
 
## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L222
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/ERC4626-Cloned.sol#L324

## Tool used

Manual Review

## Recommendation

Override the mint function and add the `whenNotPaused` modifier

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

Causes material loss of funds. Should be high risk

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Causes material loss of funds. Should be high risk

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation rejected.

This issue depends on the pausable state being active, also, it has the potential to lead to a loss of funds, it's not a guarantee. 

**sherlock-admin**

> Escalation rejected.
> 
> This issue depends on the pausable state being active, also, it has the potential to lead to a loss of funds, it's not a guarantee. 

This issue's escalations have been rejected!

Contestants' payouts and scores will not be updated.

Auditors who escalated this issue will have their escalation amount deducted from future payouts.



# Issue M-11: Buyouts of shorter duration liens can lead to the loss of borrower funds 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/171 

## Found by 
0xRajeev

## Summary

Liens whose duration is equal to (or maybe less than) `minDurationIncrease` cannot be bought out to be replaced by newer liens with lower interest rates but the same duration. This locks the borrower out of better-termed liens, effectively resulting in the loss of their funds 
 
## Vulnerability Detail

Liens whose duration is equal to (or maybe less than) `minDurationIncrease` cannot be bought out to be replaced by newer liens with lower interest rates but the exact duration because it results in an underflow in `_getRemainingInterest()`.

Example scenario: if the strategy`liendetails.duration` is <= 14 days, then it's impossible to do a buyout of a new lien because the implemented check requires to wait `minDurationIncrease`, which is set to 14 days. However, if the buyer waits 14 days, the lien is expired, which triggers the earlier mentioned underflow.

## Impact

The borrower gets locked out of better-termed liens, effectively resulting in the loss of their funds because of extra interest paid on older liens.
 
## Code Snippet

1. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L573
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L489-L490

## Tool used

Manual Review

## Recommendation

Revisit the checking logic and minimum duration as it applies to shorter-duration loans.

## Discussion

**SantiagoGregory**

We updated buyoutLien() to check for a lower interest rate *or* higher duration.



# Issue M-12: Loan duration can exceed the end of the next epoch 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/170 

## Found by 
0xRajeev

## Summary

Loan duration can exceed the end of the next epoch, which deviates from the protocol specification.

## Vulnerability Detail

From the specs: "The duration of new loans is restricted to not exceed the end of the next epoch. For example, if a PublicVault is 15 days into a 30-day epoch, new loans must not be longer than 45 days."

However, there's no enforcement of this requirement. 

## Impact

The implementation does not adhere to the spec: Loan duration can exceed the end of the next epoch, which breaks protocol specification and therefore lead to miscalculations and potential fund loss.


## Code Snippet

1. https://docs.astaria.xyz/docs/protocol-mechanics/epochs
2. https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L146-L228

## Tool used

Manual Review

## Recommendation

Implement as per specification or revisit the specification.

# Issue M-13: First ERC4626 deposit can break share calculation 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/143 

## Found by 
pashov, joestakey, ctf\_sec, ak1, \_\_141345\_\_, neila, rvierdiiev, 0xNazgul, Jeiwan

## Summary
The first depositor of an ERC4626 vault can maliciously manipulate the share price by depositing the lowest possible amount (1 wei) of liquidity and then artificially inflating ERC4626.totalAssets.

This can inflate the base share price as high as 1:1e18 early on, which force all subsequence deposit to use this share price as a base and worst case, due to rounding down, if this malicious initial deposit front-run someone else depositing, this depositor will receive 0 shares and lost his deposited assets.

## Vulnerability Detail
Given a vault with DAI as the underlying asset:

Alice (attacker) deposits initial liquidity of 1 wei DAI via `deposit()`
Alice receives 1e18 (1 wei) vault shares
Alice transfers 1 ether of DAI via transfer() to the vault to artificially inflate the asset balance without minting new shares. The asset balance is now 1 ether + 1 wei DAI -> vault share price is now very high (= 1000000000000000000001 wei ~ 1000 * 1e18)
Bob (victim) deposits 100 ether DAI
Bob receives 0 shares
Bob receives 0 shares due to a precision issue. His deposited funds are lost.

The shares are calculated as following 
`return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());`
In case of a very high share price, due to totalAssets() > assets * supply, shares will be 0.
## Impact
`ERC4626` vault share price can be maliciously inflated on the initial deposit, leading to the next depositor losing assets due to precision issues.
## Code Snippet
https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/ERC4626-Cloned.sol#L392
## Tool used

Manual Review

## Recommendation
This is a well-known issue, Uniswap and other protocols had similar issues when supply == 0.

For the first deposit, mint a fixed amount of shares, e.g. 10**decimals()
```jsx
if (supply == 0) {
    return 10**decimals; 
} else {
    return assets.mulDivDown(supply, totalAssets());
}
```

# Issue M-14: LiquidityProvider can also lend to PrivateVault 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/141 

## Found by 
neila

## Summary
Insufficient access control for lending to PublicVault
found by [yawn-c111](https://github.com/yawn-c111)

## Vulnerability Detail
Docs says as follows
https://docs.astaria.xyz/docs/intro
> Any strategists may provide their own capital to fund these loans through their own `PrivateVaults`, and whitelisted strategists can deploy `PublicVaults` that accept funds from other liquidity providers.

However, Liquidity Providers can also lend to `PrivateVault`.

This is because `lendToVault` function is controlled by `mapping(address => address) public vaults`, which are managed by `_newVault` function and include `PrivateVault`s

This leads to unexpected atttack.

## Impact 
Unexpected liquidity providers can lend to private vaults

## Code Snippet
https://github.com/unchain-dev/2022-10-astaria-UNCHAIN/blob/main/src/AstariaRouter.sol#L324

```solidity
function lendToVault(IVault vault, uint256 amount) external whenNotPaused {
    TRANSFER_PROXY.tokenTransferFrom(
      address(WETH),
      address(msg.sender),
      address(this),
      amount
    );

    require(
      vaults[address(vault)] != address(0),
      "lendToVault: vault doesn't exist"
    );
    WETH.safeApprove(address(vault), amount);
    vault.deposit(amount, address(msg.sender));
  }
```

https://github.com/unchain-dev/2022-10-astaria-UNCHAIN/blob/main/src/AstariaRouter.sol#L500

```solidity
function _newVault(
    uint256 epochLength,
    address delegate,
    uint256 vaultFee
  ) internal returns (address) {
    uint8 vaultType;

    address implementation;
    if (epochLength > uint256(0)) {
      require(
        epochLength >= minEpochLength && epochLength <= maxEpochLength,
        "epochLength must be greater than or equal to MIN_EPOCH_LENGTH and less than MAX_EPOCH_LENGTH"
      );
      implementation = VAULT_IMPLEMENTATION;
      vaultType = uint8(VaultType.PUBLIC);
    } else {
      implementation = SOLO_IMPLEMENTATION;
      vaultType = uint8(VaultType.SOLO);
    }

    //immutable data
    address vaultAddr = ClonesWithImmutableArgs.clone(
      implementation,
      abi.encodePacked(
        address(msg.sender),
        address(WETH),
        address(COLLATERAL_TOKEN),
        address(this),
        address(COLLATERAL_TOKEN.AUCTION_HOUSE()),
        block.timestamp,
        epochLength,
        vaultType,
        vaultFee
      )
    );

    //mutable data
    VaultImplementation(vaultAddr).init(
      VaultImplementation.InitParams(delegate)
    );

    vaults[vaultAddr] = msg.sender;

    emit NewVault(msg.sender, vaultAddr);

    return vaultAddr;
  }
```

## Tool used
Manual Review

## Recommendation
create requirement to lend to only PublicVaults.

# Issue M-15: LiquidationAccountant.claim may revert for some tokens 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/83 

## Found by 
0x4141

## Summary
`LiquidationAccountant.claim` may initiate a transfer with the amount 0, which reverts for some tokens.

## Vulnerability Detail
Some tokens (e.g., LEND -> see https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers) revert when a transfer with amount 0 is initiated. This can happen within `claim` when the `withdrawRatio` is 100%.

## Impact
In such a scenario, the funds are not claimable, leading to a loss of funds.

## Code Snippet
https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LiquidationAccountant.sol#L88

## Tool used

Manual Review

## Recommendation
Do not initiate a transfer when the amount is zero.

# Issue M-16: LienToken._payment function increases users debt 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/73 

## Found by 
rvierdiiev

## Summary
LienToken._payment function increases users debt by setting `lien.amount = _getOwed(lien)`
## Vulnerability Detail
`LienToken._payment` is used by `LienToken.makePayment` [function](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L387-L389) that allows borrower to repay part or all his debt.

Also this function can be called by `AuctionHouse` when the lien is liquidated.

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L594-L649
```solidity
  function _payment(
    uint256 collateralId,
    uint8 position,
    uint256 paymentAmount,
    address payer
  ) internal returns (uint256) {
    if (paymentAmount == uint256(0)) {
      return uint256(0);
    }


    uint256 lienId = liens[collateralId][position];
    Lien storage lien = lienData[lienId];
    uint256 end = (lien.start + lien.duration);
    require(
      block.timestamp < end || address(msg.sender) == address(AUCTION_HOUSE),
      "cannot pay off an expired lien"
    );


    address lienOwner = ownerOf(lienId);
    bool isPublicVault = IPublicVault(lienOwner).supportsInterface(
      type(IPublicVault).interfaceId
    );


    lien.amount = _getOwed(lien);


    address payee = getPayee(lienId);
    if (isPublicVault) {
      IPublicVault(lienOwner).beforePayment(lienId, paymentAmount);
    }
    if (lien.amount > paymentAmount) {
      lien.amount -= paymentAmount;
      lien.last = block.timestamp.safeCastTo32();
      // slope does not need to be updated if paying off the rest, since we neutralize slope in beforePayment()
      if (isPublicVault) {
        IPublicVault(lienOwner).afterPayment(lienId);
      }
    } else {
      if (isPublicVault && !AUCTION_HOUSE.auctionExists(collateralId)) {
        // since the openLiens count is only positive when there are liens that haven't been paid off
        // that should be liquidated, this lien should not be counted anymore
        IPublicVault(lienOwner).decreaseEpochLienCount(
          IPublicVault(lienOwner).getLienEpoch(end)
        );
      }
      //delete liens
      _deleteLienPosition(collateralId, position);
      delete lienData[lienId]; //full delete


      _burn(lienId);
    }


    TRANSFER_PROXY.tokenTransferFrom(WETH, payer, payee, paymentAmount);


    emit Payment(lienId, paymentAmount);
    return paymentAmount;
  }
```

The main problem is in line 617. `lien.amount = _getOwed(lien);`
https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L617

Here lien.amount becomes lien.amount + accrued interests, because `_getOwed` do that [calculation](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L549).

`lien.amount` is the amount that user borrowed. So actually that line has just increased user's debt. And in case if he didn't pay all amount of lien, then next time he will pay more interests. 

Example.
User borrows 1 eth. His `lien.amount` is 1eth.
Then he wants to repay some part(let's say 0.5 eth). Now his `lien.amount` becomes `lien.amount + interests`.
When he pays next time, he pays `(lien.amount + interests) + new interests`. So interests are acummulated on previous interests.
## Impact
User borrowed amount increases and leads to lose of funds.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Do not update lien.amount to _getOwed(lien).

## Discussion

**Evert0x**

Downgrading to medium

**IAmTurnipBoy**

Escalate for 1 USDC

Invalid. All I see here is compounding interest. Nothing incorrect here, just a design choice. Plenty of loans use compounding interest.

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Invalid. All I see here is compounding interest. Nothing incorrect here, just a design choice. Plenty of loans use compounding interest.

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation rejected

The impact of this is it'll cause a user in this situation to pay compound interest. Since loan is only 2 weeks, that will be very small impact. But the more often you make a payment the more interest you pay, a valid medium severity finding.

**sherlock-admin**

> Escalation rejected
> 
> The impact of this is it'll cause a user in this situation to pay compound interest. Since loan is only 2 weeks, that will be very small impact. But the more often you make a payment the more interest you pay, a valid medium severity finding.

This issue's escalations have been rejected!

Watsons who escalated this issue will have their escalation amount deducted from their next payout.



# Issue M-17: Any public vault without a delegate can be drained 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/69 

## Found by 
zzykxx, obront, cryptphi, HonorLt, yixxas, rvierdiiev

## Summary

If a public vault is created without a delegate, delegate will have the value of `address(0)`. This is also the value returned by `ecrecover` for invalid signatures (for example, if v is set to a position number that is not 27 or 28), which allows a malicious actor to cause the signature validation to pass for arbitrary parameters, allowing them to drain a vault using a worthless NFT as collateral.

## Vulnerability Detail

When a new Public Vault is created, the Router calls the `init()` function on the vault as follows:

```solidity
VaultImplementation(vaultAddr).init(
  VaultImplementation.InitParams(delegate)
);
```
If a delegate wasn't set, this will pass `address(0)` to the vault. If this value is passed, the vault simply skips the assignment, keeping the delegate variable set to the default 0 value:

```solidity
if (params.delegate != address(0)) {
  delegate = params.delegate;
}
```
Once the delegate is set to the zero address, any commitment can be validated, even if the signature is incorrect. This is because of a quirk in `ecrecover` which returns `address(0)` for invalid signatures. A signature can be made invalid by providing a positive integer that is not 27 or 28 as the `v` value. The result is that the following function call assigns `recovered = address(0)`:

```solidity
    address recovered = ecrecover(
      keccak256(
        encodeStrategyData(
          params.lienRequest.strategy,
          params.lienRequest.merkle.root
        )
      ),
      params.lienRequest.v,
      params.lienRequest.r,
      params.lienRequest.s
    );
```
To confirm the validity of the signature, the function performs two checks:
```solidity
require(
  recovered == params.lienRequest.strategy.strategist,
  "strategist must match signature"
);
require(
  recovered == owner() || recovered == delegate,
  "invalid strategist"
);
```
These can be easily passed by setting the `strategist` in the params to `address(0)`. At this point, all checks will pass and the parameters will be accepted as approved by the vault.

With this power, a borrower can create params that allow them to borrow the vault's full funds in exchange for a worthless NFT, allowing them to drain the vault and steal all the user's funds.

## Impact

All user's funds held in a vault with no delegate set can be stolen.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L537-L539

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L118-L124

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L167-L185

## Tool used

Manual Review, Foundry

## Recommendation

Add a require statement that the recovered address cannot be the zero address:

```solidity
require(recovered != address(0));
```

## Discussion

**sherlock-admin**

> Escalate for 1 USDC
> 
> Disagree with high. Requires a external factors and a bit of phishing to make this work. User would have to voluntarily make a lien position that has strategist == address(0) which should raise red flags. Medium seems more fitting. 
> 
> Relevant lines:
> 
> https://github.com/sherlock-audit/2022-10-astaria/blob/7d12a5516b7c74099e1ce6fb4ec87c102aec2786/src/VaultImplementation.sol#L178-L181
> 
> 

You've deleted an escalation for this issue.



# Issue M-18: LienToken.createLien doesn't check if user should be liquidated and provides new loan 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/66 

## Found by 
rvierdiiev

## Summary
`LienToken.createLien` doesn't check if user should be liquidated and provides new loan if auction do not exist for collateral.
## Vulnerability Detail
`LienToken.createLien` [relies](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L244-L246)  on `AuctionHouse` to check if new loan can be added to borrower. It assumes that if auction doesn't exist then user is safe to take new loan.

The problem is that to start auction with token that didn't pay the debt someone should call `AstariaRouter.liquidate` function. If no one did it then auction for the NFT will not exists, and `LienToken.createLien` will create new Lien to user, while he already didn't pay debt and should be liquidated.
## Impact
New loan will be paid to user that didn't repay previous lien.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Check if user can be liquidated through all of his liens positions. If not then only proceed with new loan.

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

Disagree with high severity. User would have to voluntarily sign a lien for another user who is already late on their payments (should raise some red flags), should only be medium or low given external circumstances

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Disagree with high severity. User would have to voluntarily sign a lien for another user who is already late on their payments (should raise some red flags), should only be medium or low given external circumstances

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted.

**sherlock-admin**

> Escalation accepted.

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.



# Issue M-19: Strategist nonce is not checked 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/59 

## Found by 
HonorLt, rvierdiiev, ctf\_sec, cryptphi

## Summary
Strategist nonce is not checked while checking commitment. This makes impossible for strategist to cancel signed commitment.
## Vulnerability Detail
`VaultImplementation.commitToLien` is created to give the ability to borrow from the vault. The conditions of loan are discussed off chain and owner or delegate of the vault then creates and signes deal details. Later borrower can provide it as `IAstariaRouter.Commitment calldata params` param to VaultImplementation.commitToLien.

After the checking of signer of commitment `VaultImplementation._validateCommitment` function [calls](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L187-L189) `AstariaRouter.validateCommitment`.

```solidity
  function validateCommitment(IAstariaRouter.Commitment calldata commitment)
    public
    returns (bool valid, IAstariaRouter.LienDetails memory ld)
  {
    require(
      commitment.lienRequest.strategy.deadline >= block.timestamp,
      "deadline passed"
    );


    require(
      strategyValidators[commitment.lienRequest.nlrType] != address(0),
      "invalid strategy type"
    );


    bytes32 leaf;
    (leaf, ld) = IStrategyValidator(
      strategyValidators[commitment.lienRequest.nlrType]
    ).validateAndParse(
        commitment.lienRequest,
        COLLATERAL_TOKEN.ownerOf(
          commitment.tokenContract.computeId(commitment.tokenId)
        ),
        commitment.tokenContract,
        commitment.tokenId
      );


    return (
      MerkleProof.verifyCalldata(
        commitment.lienRequest.merkle.proof,
        commitment.lienRequest.merkle.root,
        leaf
      ),
      ld
    );
  }
```

This function check additional params, one of which is `commitment.lienRequest.strategy.deadline`. But it doesn't check for the [nonce](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L77) of strategist here. But this nonce is used while [signing](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L100).

Also `AstariaRouter` gives ability to [increment nonce](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/AstariaRouter.sol#L130-L132) for strategist, but it is never called. That means that currently strategist use always same nonce and can't cancel his commitment.
## Impact
Strategist can't cancel his commitment. User can use this commitment to borrow up to 5 times.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Give ability to strategist to call `increaseNonce` function.

# Issue M-20: _validateCommitment fails for approved operators 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/52 

## Found by 
obront, rvierdiiev

## Summary

If a collateral token owner approves another user as an operator for all their tokens (rather than just for a given token), the validation check in `_validateCommitment()` will fail.

## Vulnerability Detail

The collateral token is implemented as an ERC721, which has two ways to approve another user:
- Approve them to take actions with a given token (`approve()`)
- Approve them as an "operator" for all your owned tokens (`setApprovalForAll()`)

However, when the `_validateCommitment()` function checks that the token is owned or approved by `msg.sender`, it does not accept those who are set as operators.

```solidity
if (msg.sender != holder) {
  require(msg.sender == operator, "invalid request");
}
```

## Impact

Approved operators of collateral tokens will be rejected from taking actions with those tokens.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/VaultImplementation.sol#L152-L158

## Tool used

Manual Review

## Recommendation

Include an additional check to confirm whether the `msg.sender` is approved as an operator on the token:

```solidity
    address holder = ERC721(COLLATERAL_TOKEN()).ownerOf(collateralId);
    address approved = ERC721(COLLATERAL_TOKEN()).getApproved(collateralId);
    address operator = ERC721(COLLATERAL_TOKEN()).isApprovedForAll(holder);

    if (msg.sender != holder) {
      require(msg.sender == operator || msg.sender == approved, "invalid request");
    }
```

# Issue M-21: timeToEpochEnd calculates backwards, breaking protocol math 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/50 

## Found by 
0xNazgul, 0xRajeev, obront, hansfriese

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

# Issue M-22: Strategists are paid 10x the vault fee because of a math error 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/49 

## Found by 
obront

## Summary

Strategists set their vault fee in BPS (x / 10,000), but are paid out as x / 1,000. The result is that strategists will always earn 10x whatever vault fee they set.

## Vulnerability Detail

Whenever any payment is made towards a public vault, `beforePayment()` is called, which calls `_handleStrategistInterestReward()`.

The function is intended to take the amount being paid, adjust by the vault fee to get the fee amount, and convert that amount of value into shares, which are added to `strategistUnclaimedShares`.

```solidity
function _handleStrategistInterestReward(uint256 lienId, uint256 amount)
    internal
    virtual
    override
  {
    if (VAULT_FEE() != uint256(0)) {
      uint256 interestOwing = LIEN_TOKEN().getInterest(lienId);
      uint256 x = (amount > interestOwing) ? interestOwing : amount;
      uint256 fee = x.mulDivDown(VAULT_FEE(), 1000);
      strategistUnclaimedShares += convertToShares(fee);
    }
  }
```
Since the vault fee is stored in basis points, to get the vault fee, we should take the amount, multiply it by `VAULT_FEE()` and divide by 10,000. However, we accidentally divide by 1,000, which results in a 10x larger reward for the strategist than intended.

As an example, if the vault fee is intended to be 10%, we would set `VAULT_FEE = 1000`. In that case, for any amount paid off, we would calculate `fee = amount * 1000 / 1000` and the full amount would be considered a fee for the strategist.

## Impact

Strategists will be paid 10x the agreed upon rate for their role, with the cost being borne by users.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L513-L524

## Tool used

Manual Review

## Recommendation

Change the `1000` in the `_handleStrategistInterestReward()` function to `10_000`.

## Discussion

**IAmTurnipBoy**

Escalate for 1 USDC

Don't agree with high severity. Fee can easily be changed by protocol after the fact to fix this, which is why medium makes more sense. Simple to fix as a parameter change.

**sherlock-admin**

 > Escalate for 1 USDC
> 
> Don't agree with high severity. Fee can easily be changed by protocol after the fact to fix this, which is why medium makes more sense. Simple to fix as a parameter change.

You've created a valid escalation for 1 USDC!

To remove the escalation from consideration: Delete your comment.
To change the amount you've staked on this escalation: Edit your comment **(do not create a new comment)**.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Evert0x**

Escalation accepted.


**sherlock-admin**

> Escalation accepted.
> 

This issue's escalations have been accepted!

Contestants' payouts and scores will be updated according to the changes made on this issue.

**zobront**

@Evert0x @IAmTurnipBoy VAULT_FEE is set as an immutable arg on the Vault, so there is no ability to change it later. It's permanently set and they'd need to redeploy the whole protocol to change it. This is clearly a high.



# Issue M-23: Underlying With Non-Standard Decimals Not Supported 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/33 

## Found by 
0x0

## Summary

Arithmetic operations are performed with the assumption that the token always has 18 decimals.

## Vulnerability Detail

[`LiquidationAccountant.claim`
](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LiquidationAccountant.sol#L65)

Arithmetic operations assume the token has [18 decimals](https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LiquidationAccountant.sol#L78). Not all tokens use 18 decimals, such as Tether.

## Impact

- The addition of underlying capital that does not use 18 decimals will not be possible.

## Code Snippet

```solidity
uint256 transferAmount = withdrawRatio.mulDivDown(balance, 1e18);
```

## Tool used

Manual Review

## Recommendation

- Consider whether the addition of capital that does not use 18 decimals is desirable in the future. If it is, refactor contracts to support tokens with non-standard decimals.

# Issue M-24: _payment() function transfers full paymentAmount, overpaying first liens 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/28 

## Found by 
tives, ak1, obront, \_\_141345\_\_, bin2chen, 0xRajeev, hansfriese, rvierdiiev

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

# Issue M-25: _getInterest() function uses block.timestamp instead of the inputted timestamp 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/25 

## Found by 
obront, sorrynotsorry

## Summary

The `_getInterest()` function takes a timestamp as input. However, in a crucial check in the function, it uses `block.timestamp` instead. The result is that other functions expecting accurate interest amounts will receive incorrect values.

## Vulnerability Detail

The `_getInterest()` function takes a lien and a timestamp as input. The intention is for it to calculate the amount of time that has passed in the lien (`delta_t`) and multiply this value by the rate and the amount to get the interest generated by this timestamp.

However, the function uses the following check regarding the timestamp:

```solidity
if (block.timestamp >= lien.start + lien.duration) {
  delta_t = uint256(lien.start + lien.duration - lien.last);
} 
```

Because this check uses `block.timestamp` before returning the maximum interest payment, the function will incorrectly determine which path to take, and return an incorrect interest value.

## Impact

There are two negative consequences that can come from this miscalculation:

- if the function is called when the lien is over (`block.timestamp >= lien.start + lien.duration`) to check an interest amount from a timestamp during the lien, it will incorrectly return the maximum interest value
- If the function is called when the lien is active for a timestamp long after the lien is over, it will skip the check to return maximum value and return the value that would have been generated if interest kept accruing indefinitely (using `delta_t = uint256(timestamp.safeCastTo32() - lien.last);`)

This `_getInterest()` function is used in many crucial protocol functions (`_getOwed()`, `calculateSlope()`, `changeInSlope()`, `getTotalDebtForCollateralToken()`), so these incorrect values can have surprising and unexpected negative impacts on the protocol.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/LienToken.sol#L177-L196

## Tool used

Manual Review

## Recommendation

Change `block.timestamp` to `timestamp` so that the if statement checks correctly.

# Issue M-26: Vault Fee uses incorrect offset leading to wildly incorrect value, allowing strategists to steal all funds 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/19 

## Found by 
obront, zzykxx

## Summary

`VAULT_FEE()` uses an incorrect offset, returning a number ~1e16X greater than intended, providing strategists with unlimited access to drain all vault funds.

## Vulnerability Detail

When using ClonesWithImmutableArgs, offset values are set so that functions representing variables can retrieve the correct values from storage. 

In the ERC4626-Cloned.sol implementation, `VAULT_TYPE()` is given an offset of 172. However, the value before it is a `uint8` at the offset 164. Since a `uint8` takes only 1 byte of space, `VAULT_TYPE()` should have an offset of 165.

I put together a POC to grab the value of `VAULT_FEE()` in the test setup:

```solidity
function testVaultFeeIncorrectlySet() public {
  Dummy721 nft = new Dummy721();
  address tokenContract = address(nft);
  uint256 tokenId = uint256(1);
  address publicVault = _createPublicVault({
      strategist: strategistOne,
      delegate: strategistTwo,
      epochLength: 14 days
  });
  uint fee = PublicVault(publicVault).VAULT_FEE();
  console.log(fee)
  assert(fee == 5000); // 5000 is the value that was meant to be set
}
```

In this case, the value returned is > 3e20. 

## Impact

This is a highly critical bug. `VAULT_FEE()` is used in `_handleStrategistInterestReward()` to determine the amount of tokens that should be allocated to `strategistUnclaimedShares`.

```solidity
if (VAULT_FEE() != uint256(0)) {
    uint256 interestOwing = LIEN_TOKEN().getInterest(lienId);
    uint256 x = (amount > interestOwing) ? interestOwing : amount;
    uint256 fee = x.mulDivDown(VAULT_FEE(), 1000); //VAULT_FEE is a basis point
    strategistUnclaimedShares += convertToShares(fee);
  }
```
The result is that strategistUnclaimedShares will be billions of times higher than the total interest generated, essentially giving strategist access to withdraw all funds from their vaults at any time.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/ERC4626-Cloned.sol#L113-L116

https://github.com/sherlock-audit/2022-10-astaria/blob/main/src/PublicVault.sol#L518-L523

## Tool used

Manual Review

## Recommendation

Set the offset for `VAULT_FEE()` to 165. I tested this value in the POC I created and it correctly returned the value of 5000.

# Issue M-27: Bids cannot be created within timeBuffer of completion of a max duration auction 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/18 

## Found by 
rvierdiiev, minhquanym, 0x4141, obront, hansfriese, TurnipBoy, peanuts, yixxas, neila, Prefix, csanuragjain, 0xRajeev, Jeiwan

## Summary

The auction mechanism is intended to watch for bids within `timeBuffer` of the end of the auction, and automatically increase the remaining duration to `timeBuffer` if such a bid comes in.

There is an error in the implementation that causes all bids within `timeBuffer` of the end of a max duration auction to revert, effectively ending the auction early and cutting off bidders who intended to wait until the end.

## Vulnerability Detail

In the `createBid()` function in AuctionHouse.sol, the function checks if a bid is within the final `timeBuffer` of the auction:

```solidity
if (firstBidTime + duration - block.timestamp < timeBuffer)
```

If so, it sets `newDuration` to equal the amount that will extend the auction to `timeBuffer` from now:

```solidity
uint64 newDuration = uint256( duration + (block.timestamp + timeBuffer - firstBidTime) ).safeCastTo64();
```

If this `newDuration` doesn't extend beyond the `maxDuration`, this works great. However, if it does extend beyond `maxDuration`, the following code is used to update `duration`:

```solidity
auctions[tokenId].duration = auctions[tokenId].maxDuration - firstBidTime;
```

This code is incorrect. `maxDuration` will be a duration for the contest (currently set to 3 days), whereas `firstTimeBid` is a timestamp for the start of the auction (current timestamps are > 1 billion). 

Subtracting `firstTimeBid` from `maxDuration` will underflow, which will revert the function.

## Impact

- Bidders who expected to wait until the end of the auction to vote will be cut off from voting, as the auction will revert their bids.
- Vaults whose collateral is up for auction will earn less than they otherwise would have.

## Code Snippet

https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/AuctionHouse.sol#L127-L146

## Tool used

Manual Review

## Recommendation

Change this assignment to simply assign `duration` to `maxDuration`, as follows:

```solidity
auctions[tokenId].duration = auctions[tokenId].maxDuration
```

# Issue M-28: ERC4626 does not work with fee-on-transfer tokens 

Source: https://github.com/sherlock-audit/2022-10-astaria-judging/issues/3 

## Found by 
pashov, w42d3n, Bnke0x0

## Summary

## Vulnerability Detail

## Impact
The ERC4626-Cloned.deposit/mint functions do not work well with fee-on-transfer tokens as the `assets` variable is the pre-fee amount, including the fee, whereas the totalAssets do not include the fee anymore.

## Code Snippet
This can be abused to mint more shares than desired.

https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/ERC4626-Cloned.sol#L305-L322

             '  function deposit(uint256 assets, address receiver)
                 public
                 virtual
                 override(IVault)
                 returns (uint256 shares)
               {
                 // Check for rounding error since we round down in previewDeposit.
                 require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

                 // Need to transfer before minting or ERC777s could reenter.
                 ERC20(underlying()).safeTransferFrom(msg.sender, address(this), assets);

                 _mint(receiver, shares);

                 emit Deposit(msg.sender, receiver, assets, shares);

                 afterDeposit(assets, shares);
               }'

https://github.com/sherlock-audit/2022-10-astaria/blob/main/lib/astaria-gpl/src/ERC4626-Cloned.sol#L315

     `ERC20(underlying()).safeTransferFrom(msg.sender, address(this), assets);`

A `deposit(1000)` should result in the same shares as two deposits of `deposit(500)` but it does not because `assets` is the pre-fee amount.
Assume a fee-on-transfer of `20%`. Assume current `totalAmount = 1000`, `totalShares = 1000` for simplicity.

`deposit(1000) = 1000 / totalAmount * totalShares = 1000 shares`.
`deposit(500) = 500 / totalAmount * totalShares = 500 shares`. Now the `totalShares` increased by 500 but the `totalAssets` only increased by `(100% - 20%) * 500 = 400`. Therefore, the second `deposit(500) = 500 / (totalAmount + 400) * (newTotalShares) = 500 / (1400) * 1500 = 535.714285714 shares`.

In total, the two deposits lead to `35` more shares than a single deposit of the sum of the deposits.

## Tool used

Manual Review

## Recommendation
`assets` should be the amount excluding the fee, i.e., the amount the contract actually received.
This can be done by subtracting the pre-contract balance from the post-contract balance.
However, this would create another issue with ERC777 tokens.

Maybe `previewDeposit` should be overwritten by vaults supporting fee-on-transfer tokens to predict the post-fee amount. And do the shares computation on that, but then the `afterDeposit` is still called with the original `assets`and implementers need to be aware of this.

## Discussion

**androolloyd**

we will not be supporting fee on transfer tokens at the time



