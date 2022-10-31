obront

high

# Any public vault without a delegate can be drained

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