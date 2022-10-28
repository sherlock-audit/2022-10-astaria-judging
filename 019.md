obront

high

# Vault Fee uses incorrect offset leading to wildly incorrect value, allowing strategists to steal all funds

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