WATCHPUG

high

# Wrong Oracle feed addresses

## Summary

Wrong Oracle feed addresses will result in wrong prices.

## Vulnerability Detail

StableOracleWBTC.sol#L17 the address is not the BTC/USD feed address.

StableOracleDAI.sol#L28, `DAIEthOracle` is wrong.

StableOracleDAI.sol#L30, address for `ethOracle` is address zero (a hanging todo).

StableOracleWBGL.sol#L19, the address for staticOracleUniV3 is wrong, the current one is actually the univ3 pool address.

## Impact

Wrong prices for collateral assets.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L8-L28

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L23-L31

https://github.com/USSDofficial/ussd-contracts/blob/f44c726371f3152634bcf0a3e630802e39dec49c/contracts/oracles/StableOracleWBGL.sol#L17-L22

## Tool used

Manual Review

## Recommendation

Use correct addresses.