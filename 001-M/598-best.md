RaymondFam

medium

# Risk of Incorrect Asset Pricing by StableOracle in Case of Underlying Aggregator Reaching minAnswer

## Summary
Chainlink aggregators have a built-in circuit breaker to prevent the price of an asset from deviating outside a predefined price range. This circuit breaker may cause the oracle to persistently return the minPrice instead of the actual asset price in the event of a significant price drop, as witnessed during the LUNA crash. 

## Vulnerability Detail
StableOracleDAI.sol, StableOracleWBTC.sol, and StableOracleWETH.sol utilize the ChainlinkFeedRegistry to fetch the price of the requested tokens.

```solidity
function latestRoundData(
  address base,
  address quote
)
  external
  view
  override
  checkPairAccess()
  returns (
    uint80 roundId,
    int256 answer,
    uint256 startedAt,
    uint256 updatedAt,
    uint80 answeredInRound
  )
{
  uint16 currentPhaseId = s_currentPhaseId[base][quote];
  AggregatorV2V3Interface aggregator = _getFeed(base, quote);
  require(address(aggregator) != address(0), "Feed not found");
  (
    roundId,
    answer,
    startedAt,
    updatedAt,
    answeredInRound
  ) = aggregator.latestRoundData();
  return _addPhaseIds(roundId, answer, startedAt, updatedAt, answeredInRound, currentPhaseId);
}
```
ChainlinkFeedRegistry#latestRoundData extracts the linked aggregator and requests round data from it. If an asset's price falls below the minPrice, the protocol continues to value the token at the minPrice rather than its real value. This discrepancy could have the protocol end up [minting](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L151-L173) drastically larger amount of stableCoinAmount as well as returning a much bigger [collateral factor](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L179-L194).

For instance, if TokenA's minPrice is $1 and its price falls to $0.10, the aggregator continues to report $1, rendering the related function calls to entail a value that is ten times the actual value.

It's important to note that while Chainlink oracles form part of the OracleAggregator system and the use of a combination of oracles could potentially prevent such a situation, there's still a risk. Secondary oracles, such as Band, could potentially be exploited by a malicious user who can DDOS relayers to prevent price updates. Once the price becomes stale, the Chainlink oracle's price would be the sole reference, posing a significant risk.

## Impact
In the event of an asset crash (like LUNA), the protocol can be manipulated to handle calls at an inflated price.

## Code Snippet
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L33-L53
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L21-L26
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWETH.sol#L21-L26

## Tool used

Manual Review

## Recommendation
StableOracle should cross-check the returned answer against the minPrice/maxPrice and revert if the answer is outside of these bounds:

```solidity
    (, int256 price, , uint256 updatedAt, ) = registry.latestRoundData(
        token,
        USD
    );
    
    if (price >= maxPrice or price <= minPrice) revert();
```
This ensures that a false price will not be returned if the underlying asset's value hits the minPrice.