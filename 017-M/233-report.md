TurnipBoy

high

# _deleteLienPosition can be called by anyone to delete any lien they wish

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