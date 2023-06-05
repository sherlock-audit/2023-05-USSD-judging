Proxy

medium

# Not using slippage parameter or deadline while swapping on UniswapV3

## Summary

While making a swap on UniswapV3 the caller should use the slippage parameter `amountOutMinimum` and `deadline` parameter to avoid losing funds.

## Vulnerability Detail

[`UniV3SwapInput()`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L227-L240) in `USSD` contract does not use the slippage parameter [`amountOutMinimum`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L237)  nor [`deadline`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L235). 

`amountOutMinimum` is used to specify the minimum amount of tokens the caller wants to be returned from a swap. Using `amountOutMinimum = 0` tells the swap that the caller will accept a minimum amount of 0 output tokens from the swap, opening up the user to a catastrophic loss of funds via [MEV bot sandwich attacks](https://medium.com/coinmonks/defi-sandwich-attack-explain-776f6f43b2fd). 

`deadline` lets the caller specify a deadline parameter that enforces a time limit by which the transaction must be executed. Without a deadline parameter, the transaction may sit in the mempool and be executed at a much later time potentially resulting in a worse price for the user.

## Impact

Loss of funds and not getting the correct amount of tokens in return.

## Code Snippet

- Function [`UniV3SwapInput()`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L227-L240)
  - Not using [`amountOutMinimum`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L237)
  - Not using [`deadline`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L235)


## Tool used

Manual Review

## Recommendation

Use parameters `amountOutMinimum` and `deadline` correctly to avoid loss of funds.