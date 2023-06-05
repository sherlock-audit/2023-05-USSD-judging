J4de

medium

# `USSD.sol#initialize` mint without deposit collateral

## Summary

`USSD.sol#initialize` mint 10000 USSD without deposit any collateral

## Vulnerability Detail

The described in the whitepaper:

> USSD coin is ’initialized’ with one-time minting of 1000 USSD, taking 1000
> DAI as collateral.

But in the implementation, the `initialize` function does not deposit any DAI. And mint 10000 USSD instead of the 1000 USSD mentioned in the whitepaper.

## Impact

Does not meet the white paper description, and may cause the initial state of USSD to be unstable (lack of sufficient collateral to resist depeg)

## Code Snippet

https://github.com/USSDofficial/ussd-contracts/blob/f44c726371f3152634bcf0a3e630802e39dec49c/contracts/USSD.sol#L42

## Tool used

Manual Review

## Recommendation

It is recommended to mint 1000 USSD with 1000 DAI