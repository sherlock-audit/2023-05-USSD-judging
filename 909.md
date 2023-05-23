WATCHPUG

high

# Oracle price should be denominated in DAI instead of USD

## Summary

## Vulnerability Detail

Per the whitepaper, USSD aims to be pegged to DAI.

The implementation of rebalancer is also using the DAI price of USSD (`getOwnValuation()`) as the target of the peg.

However, all current oracles return the price denominated in USD.

Additionally, all collateral tokens can be used to mint at the oracle price with no fee.

As a result, when DAI is over-pegged, the system will automatically drive itself away from the peg to DAI. The proof of concept for this is as follows:

When the DAI price is 1.1 (over-pegged), and the system is actively maintaining its peg to DAI, the user can:

1. Mint 1100 USSD with 1000 DAI (worth 1100 USD).
2. Sell 1100 USSD for 1100 DAI, driving the USSD to a lower price against DAI, say 1 USSD to 0.98 DAI.
3. Trigger the `rebalance()` function which sells collateral to DAI and buys back USSD to push its price back to 1:1 with DAI.

By repeating steps 1-3, the system will consume all its collateral, and the profit will be extracted by the arbitrator.

## Impact

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWETH.sol#L12-L27

## Tool used

Manual Review

## Recommendation

Change the unit of Oracle price from USD to DAI.

- DAI should return 1
- WBTC should use WBTC/ETH ETH/DAI
- WETH should use WETH/DAI
- WBGL should use WBGL/WETH ETH/DAI

Or, change the peg target from DAI to USD, which means the `getOwnValuation()` should not be used as the peg deviation check standard.