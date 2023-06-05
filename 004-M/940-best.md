GimelSec

medium

# `USSDRebalancer.SellUSSDBuyCollateral` should revert if `flutter == flutterRatios.length`

## Summary

`USSDRebalancer.SellUSSDBuyCollateral` iterates `flutterRatios` to find out which flutter is matched. But if none is matched, `USSDRebalancer.SellUSSDBuyCollateral` doesn’t revert.

## Vulnerability Detail

Even If none of the flutters is matched, `USSDRebalancer.SellUSSDBuyCollateral` still executes the rest of the code. But it could lead to unexpected results.
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L180
```solidity
    function BuyUSSDSellCollateral(uint256 amountToBuy) internal {
      …

      // total collateral portions
      uint256 cf = IUSSD(USSD).collateralFactor();
      uint256 flutter = 0;
      for (flutter = 0; flutter < flutterRatios.length; flutter++) {
        if (cf < flutterRatios[flutter]) {
          break;
        }
      }

      CollateralInfo[] memory collateral = IUSSD(USSD).collateralList();
      uint portions = 0;
      uint ownval = (getOwnValuation() * 1e18 / 1e6) * IUSSD(USSD).totalSupply() / 1e6; // 1e18 total USSD value
      for (uint256 i = 0; i < collateral.length; i++) {
        uint256 collateralval = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * 1e18 / (10**IERC20MetadataUpgradeable(collateral[i].token).decimals()) * collateral[i].oracle.getPriceUSD() / 1e18;
        if (collateralval * 1e18 / ownval < collateral[i].ratios[flutter]) {
          portions++;
        }
      }

      …
    }
```

## Impact

If none of the flutters is matched, it could lead to unexpected results.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L180


## Tool used

Manual Review

## Recommendation

If `flutter == flutterRatios.length`, it should revert.
```diff
    function BuyUSSDSellCollateral(uint256 amountToBuy) internal {
      …

      // total collateral portions
      uint256 cf = IUSSD(USSD).collateralFactor();
      uint256 flutter = 0;
      for (flutter = 0; flutter < flutterRatios.length; flutter++) {
        if (cf < flutterRatios[flutter]) {
          break;
        }
      }
+     require(flutter != flutterRatios.length, "no flutter matched");

      CollateralInfo[] memory collateral = IUSSD(USSD).collateralList();
      uint portions = 0;
      uint ownval = (getOwnValuation() * 1e18 / 1e6) * IUSSD(USSD).totalSupply() / 1e6; // 1e18 total USSD value
      for (uint256 i = 0; i < collateral.length; i++) {
        uint256 collateralval = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * 1e18 / (10**IERC20MetadataUpgradeable(collateral[i].token).decimals()) * collateral[i].oracle.getPriceUSD() / 1e18;
        if (collateralval * 1e18 / ownval < collateral[i].ratios[flutter]) {
          portions++;
        }
      }

      …
    }
```
