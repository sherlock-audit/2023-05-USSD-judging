saidam017

high

# rebalance process incase of  selling the collateral, could revert because of underflow calculation

## Summary

rebalance process, will try to sell the collateral in case of peg-down. However, the process can revert because the calculation can underflow.

## Vulnerability Detail

Inside `rebalance()` call, if `BuyUSSDSellCollateral()` is triggered, it will try to sell the current collateral to `baseAsset`. The asset that will be sold (`amountToSellUnits`) first calculated. Then swap it to `baseAsset` via uniswap. However, when subtracting `amountToBuyLeftUSD`, it with result of `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)`. There is no guarantee `amountToBuyLeftUSD` always bigger than `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)`.

This causing the call could revert in case `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)` > `amountToBuyLeftUSD`.

There are two branch where `amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)` is performed : 

1. Incase `collateralval > amountToBuyLeftUSD`

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L116-L125

`collateralval` is calculated using oracle price, thus the result of swap not guaranteed to reflect the proportion of `amountToBuyLefUSD` against `collateralval` ratio, and could result in returning `baseAsset` larger than expected. And potentially  `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)` > `amountToBuyLeftUSD`

```solidity
        uint256 collateralval = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * 1e18 / (10**IERC20MetadataUpgradeable(collateral[i].token).decimals()) * collateral[i].oracle.getPriceUSD() / 1e18;
        if (collateralval > amountToBuyLeftUSD) {
          // sell a portion of collateral and exit
          if (collateral[i].pathsell.length > 0) {
            uint256 amountBefore = IERC20Upgradeable(baseAsset).balanceOf(USSD);
            uint256 amountToSellUnits = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * ((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18) / 1e18;
            IUSSD(USSD).UniV3SwapInput(collateral[i].pathsell, amountToSellUnits);
            amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
            DAItosell += (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
          } else {
```

2. Incase `collateralval < amountToBuyLeftUSD`

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L132-L138

This also can't guarantee `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)` < `amountToBuyLeftUSD`.

```solidity
          if (collateralval >= amountToBuyLeftUSD / 20) {
            uint256 amountBefore = IERC20Upgradeable(baseAsset).balanceOf(USSD);
            // sell all collateral and move to next one
            IUSSD(USSD).UniV3SwapInput(collateral[i].pathsell, IERC20Upgradeable(collateral[i].token).balanceOf(USSD));
            amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
            DAItosell += (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
          }
```

## Impact

Rebalance process can revert caused by underflow calculation.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L116-L125
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L132-L138

## Tool used

Manual Review

## Recommendation

Check if `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)` > `amountToBuyLeftUSD`, in that case, just set `amountToBuyLeftUSD` to 0.

```solidity
          ...
            uint baseAssetChange = IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
            if (baseAssetChange > amountToBuyLeftUSD) {
                amountToBuyLeftUSD = 0;
            } else {
                amountToBuyLeftUSD -= baseAssetChange;
           }
            DAItosell += baseAssetChange;
          ...
```

