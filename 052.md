obront

medium

# _validateCommitment fails for approved operators

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