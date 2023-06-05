Dug

medium

# Incorrect price feed used in `StableOracleWBTC`

## Summary

Instead of using the BTC/USD price feed, `StableOracleWBTC` is configured to use the ETH/USD price feed.

## Vulnerability Detail

The constructor of `StableOracleWBTC` currently sets the price feed as follows...

```solidity
    constructor() {
        priceFeed = AggregatorV3Interface(0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419);
    }
```

If you look up this address, you find that it is for the ETH/USD price feed. This is incorrect, as the price feed should be for BTC/USD.

https://docs.chain.link/data-feeds/price-feeds/addresses

## Impact

This means that the price returned by the oracle is incorrect, drastically undervaluing the collateral in the system. The rebalancer will not be able to function correctly when swapping underlying assets.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L15-L19

## Tool used

Manual Review

## Recommendation

The correct address for the BTC/USD price feed is `0xf4030086522a5beea4988f8ca5b36dbc97bee88c`. It should be used instead of the ETH/USD price feed.

```diff
    constructor() {
-       priceFeed = AggregatorV3Interface(0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419);
+       priceFeed = AggregatorV3Interface(0xf4030086522a5beea4988f8ca5b36dbc97bee88c);
    }
```