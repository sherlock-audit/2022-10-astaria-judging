Bnke0x0

medium

# ERC4626 does not work with fee-on-transfer tokens

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
