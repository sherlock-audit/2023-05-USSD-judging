neumo

medium

# Miscalculation of collateral factor

## Summary
If a collateral is added more than once in the `collateral` array, the calculation of the collateral factor would account for the collateral as many times as it exists in the array, making the whole calculation wrong.

## Vulnerability Detail
Function `addCollateral` allows that a collateral token is added to the `collateral` array more than once.
When calculating the collateral factor:
```solidity
function collateralFactor() public view override returns (uint256) {
	uint256 totalAssetsUSD = 0;
	for (uint256 i = 0; i < collateral.length; i++) {
		totalAssetsUSD +=
			(((IERC20Upgradeable(collateral[i].token).balanceOf(
				address(this)
			) * 1e18) /
				(10 **
					IERC20MetadataUpgradeable(collateral[i].token)
						.decimals())) *
				collateral[i].oracle.getPriceUSD()) /
			1e18;
	}

	return (totalAssetsUSD * 1e6) / totalSupply();
}
```

We see the function loops over all the collaterals present in the array, meaning that, for example, if a collateral was added twice, it will be counted two times in the calculation, when only should be counted once.

## Impact
Medium.

## Code Snippet
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L84-L108
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L179-L194

## Tool used
Manual review.


## Recommendation
Disallow duplicates in `collateral` array.
