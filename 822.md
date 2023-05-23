WATCHPUG

high

# Using WBGL as collateral poses systemic risk to the USSD stablecoin.

## Summary

WBGL has low liquidity and accepting it as collateral without limit can lead to severe depeg of the stablecoin, allowing attackers to manipulate the price and dump other collateral assets for profit. This is due to the ease of manipulating the price of WBGL.

## Vulnerability Detail

WBGL is a small-cap token with very thin liquidity.

Accepting WBGL as collateral without a cap, ie, allows users to mint as much as they want at the easily manipulated price, which can result in severe depeg of the stablecoin. 

## Impact

Specifically, an attacker could:

1. manipulate the price of WBGL to a higher price;
2. mint and dump part of the USSD for DAI;
3. trigger a rebalance, sell the collateral assets to pump the USSD price;
4. dump the rest from step 2 for more DAI.

The root cause of this issue is the thin liquidity of WBGL, which makes it easy to manipulate its price.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBGL.sol#L24-L39

## Tool used

Manual Review

## Recommendation

Remove WBGL from the list of collateral. Or, adding a cap for using WBGL as the collateral to mint USSD, say, no more than 1/10th of the liquidity of WBGL.