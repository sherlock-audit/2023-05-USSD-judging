RaymondFam

medium

# StableOracleWBTC's Dependency on BTC/USD Chainlink Oracle: Risk of Mispricing WBTC in Event of Depegging

## Summary
StableOracleWBTC.sol currently uses the BTC/USD Chainlink oracle to price Wrapped Bitcoin (WBTC). As WBTC is a bridged asset, it's value is intrinsically linked to BTC. However, in the event of a bridge compromise or failure, WBTC could depeg from BTC, rendering WBTC essentially worthless, while it would still be priced as equivalent to BTC by the protocol. 

## Vulnerability Detail
The functions `USSD.calculateMint()` and `USSD.collateralFactor()` depend on `StableOracleWBTC.getPriceUSD()`. The latter utilizes the BTC/USD Chainlink oracle to retrieve the price of WBTC.

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#LL12C1-L26C6

```solidity
contract StableOracleWBTC is IStableOracle {
    AggregatorV3Interface priceFeed;

    constructor() {
        priceFeed = AggregatorV3Interface(
            0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419
        );
    }

    function getPriceUSD() external view override returns (uint256) {
        (, int256 price, , , ) = priceFeed.latestRoundData();
        return uint256(price) * 1e10;
    }
```
## Impact
In the event of a bridge failure and WBTC depegging from BTC, the protocol would both end up minting drastically smaller amount of `stableCoinAmount` as well as returning a much smaller `collateral factor` due to its erroneous valuation of WBTC.

## Code Snippet
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L151-L173
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L179-L194

## Tool used

Manual Review

## Recommendation
A dual oracle setup is recommended for mitigating this risk. Use both the Chainlink oracle and another on-chain liquidity-based oracle (e.g., UniV3 TWAP). If the price from the liquidity oracle falls below a specific threshold compared to the Chainlink oracle (e.g., 2% lower), all related function calls should immediately revert. This dual system would use the Chainlink oracle to deter price manipulation and the liquidity oracle to protect against asset depegging.