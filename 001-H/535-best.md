0xRobocop

high

# Wrong computation of the amountToSellUnit variable

## Summary

The variable `amountToSellUnits` is computed wrongly in the code which will lead to an incorrect amount of collateral to be sold.

## Vulnerability Detail

The `BuyUSSDSellCollateral()` function is used to sell collateral during a peg-down recovery event. The computation of the amount to sell is computed using the following formula:

```solidity
// @audit-issue Wrong computation
uint256 amountToSellUnits = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * ((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18) / 1e18;
```

The idea is to sell an amount which is equivalent (in USD) to the ratio of `amountToBuyLeftUSD / collateralval`. Flattening the equation it ends up as:

```solidity
uint256 amountToSellUnits = (collateralBalance * amountToBuyLeftUSD * 1e18) / (collateralval * 1e18 * 1e18);

// Reducing the equation
uint256 amountToSellUnits = (collateralBalance * amountToBuyLeftUSD) / (collateralval * 1e18);
```

`amountToBuyLeftUSD` and `collateralval` already have 18 decimals so their decimals get cancelled together which will lead the last 1e18 factor as not necessary.

## Impact

The contract will sell an incorrect amount of collateral during a peg-down recovery event.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/6d7a9fdfb1f1ed838632c25b6e1b01748d0bafda/ussd-contracts/contracts/USSDRebalancer.sol#L121

## Tool used

Manual Review

## Recommendation

Delete the last 1e18 factor