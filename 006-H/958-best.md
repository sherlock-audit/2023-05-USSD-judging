WATCHPUG

medium

# Lack of Redeem Feature

## Summary

## Vulnerability Detail

The whitepaper mentions a redeeming feature that allows the user to redeem USSD for DAI (see section 4 "Mint and redeem"), but it is currently missing from the implementation.

Although there is a "redeem" boolean in the collateral settings, there is no corresponding feature that enables the redemption of USSD to any of the underlying collateral assets.

## Impact

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/interfaces/IUSSDRebalancer.sol#L13-L21

## Tool used

Manual Review

## Recommendation

Revise the whitepaper and docs to reflect the fact that there is no redeem function or add a redeem function.