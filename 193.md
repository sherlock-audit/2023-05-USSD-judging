J4de

high

# `USSDRebalancer.sol#SellUSSDBuyCollateral` the check of whether collateral is DAI is wrong

## Summary

The `SellUSSDBuyCollateral` function use `||` instand of `&&` to check whether the collateral is DAI. It is wrong and may cause `SellUSSDBuyCollateral` function revert.

## Vulnerability Detail

```solidity
196       for (uint256 i = 0; i < collateral.length; i++) {
197         uint256 collateralval = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * 1e18 / (10**IERC20MetadataUpgradeable(collateral[i].token).decimals()) * collateral[i].oracle.getPriceUSD() / 1e18;
198         if (collateralval * 1e18 / ownval < collateral[i].ratios[flutter]) {
199 >>        if (collateral[i].token != uniPool.token0() || collateral[i].token != uniPool.token1()) {
200             // don't touch DAI if it's needed to be bought (it's already bought)
201             IUSSD(USSD).UniV3SwapInput(collateral[i].pathbuy, daibought/portions);
202           }
203         }
204       }
```

Line 199 should use `&&` instand of `||` to ensure that the token is not DAI. If the token is DAI, the `UniV3SwapInput` function will revert because that DAI's `pathbuy` is empty.

## Impact

The `SellUSSDBuyCollateral` will revert and USSD will become unstable.

## Code Snippet

https://github.com/USSDofficial/ussd-contracts/blob/f44c726371f3152634bcf0a3e630802e39dec49c/contracts/USSDRebalancer.sol#L199

## Tool used

Manual Review

## Recommendation

```diff
      for (uint256 i = 0; i < collateral.length; i++) {
        uint256 collateralval = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * 1e18 / (10**IERC20MetadataUpgradeable(collateral[i].token).decimals()) * collateral[i].oracle.getPriceUSD() / 1e18;
        if (collateralval * 1e18 / ownval < collateral[i].ratios[flutter]) {
-         if (collateral[i].token != uniPool.token0() || collateral[i].token != uniPool.token1()) {
+         if (collateral[i].token != uniPool.token0() && collateral[i].token != uniPool.token1()) {
            // don't touch DAI if it's needed to be bought (it's already bought)
            IUSSD(USSD).UniV3SwapInput(collateral[i].pathbuy, daibought/portions);
          }
        }
      }
```