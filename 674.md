PokemonAuditSimulator

high

# Multiple wrong addresses used for oracles

## Summary
The helper contracts that `USSD` uses have implemented multiple oracles but with hard-coded wrong addresses.

## Vulnerability Detail
 - [0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L17) witch is used in **WBTC** feed, but it's for **ETH/USD**
 - [0x982152A6C7f732Ec7C9EA998dDD9Ebde00Dfa16e](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L28) witch is used in **DAI/ETH** feed, but it's for **WBGL**
## Impact
Oracles not working leading to  loss of funds or DoS in the best case scenario.
## Code Snippet
In [StableOracleDAI](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L28)
```jsx
DAIEthOracle = IStaticOracle(
       0x982152A6C7f732Ec7C9EA998dDD9Ebde00Dfa16e//@audit this is WBGL/WETH but used in DAI???
);
```
In [StableOracleWBTC](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol)
```jsx
priceFeed = AggregatorV3Interface(
       0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419//@audit isn't this ETH/USD???
);
```
## Tool used

Manual Review

## Recommendation
Implement the right addresses using
[Chainlink docs](https://docs.chain.link/data-feeds/price-feeds/addresses/)
[coinmarketcap/WBGL](https://coinmarketcap.com/dexscan/ethereum/0x982152a6c7f732ec7c9ea998ddd9ebde00dfa16e/)