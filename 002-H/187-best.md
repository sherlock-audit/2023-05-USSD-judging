J4de

medium

# `USSDRebalancer.sol#rebalance` has a loss of DAI precision

## Summary

`USSDRebalancer.sol#rebalance` has a loss of DAI precision

## Vulnerability Detail

```solidity
File: USSDRebalancer.sol
 97         BuyUSSDSellCollateral((USSDamount - DAIamount / 1e12)/2);
```

The `rebalance` function will discard the last 12 decimals of `DAIamount` then call `BuyUSSDSellCollateral` function.

```solidity
File: USSDRebalancer.sol
109     function BuyUSSDSellCollateral(uint256 amountToBuy) internal {
110       CollateralInfo[] memory collateral = IUSSD(USSD).collateralList();
111       //uint amountToBuyLeftUSD = amountToBuy * 1e12 * 1e6 / getOwnValuation();
112       uint amountToBuyLeftUSD = amountToBuy * 1e12;
```

The `BuyUSSDSellCollateral` function then multiply `amountToBuy` by `1e12`. So here will lose 12 precision and it's completely unnecessary.

## Impact

loss 12 decimals of DAI, cause USSD not rebalance as expected

## Code Snippet

https://github.com/USSDofficial/ussd-contracts/blob/f44c726371f3152634bcf0a3e630802e39dec49c/contracts/USSDRebalancer.sol#L97

## Tool used

Manual Review

## Recommendation

```diff
    function rebalance() override public {
      uint256 ownval = getOwnValuation();
      (uint256 USSDamount, uint256 DAIamount) = getSupplyProportion();
      if (ownval < 1e6 - threshold) {
        // peg-down recovery
-       BuyUSSDSellCollateral((USSDamount - DAIamount / 1e12)/2);
+       BuyUSSDSellCollateral((USSDamount * 1e12 - DAIamount)/2);
      } else if (ownval > 1e6 + threshold) {
        // mint and buy collateral
        // never sell too much USSD for DAI so it 'overshoots' (becomes more in quantity than DAI on the pool)
        // otherwise could be arbitraged through mint/redeem
        // the execution difference due to fee should be taken into accounting too
        // take 1% safety margin (estimated as 2 x 0.5% fee)
        IUSSD(USSD).mintRebalancer(((DAIamount / 1e12 - USSDamount)/2) * 99 / 100); // mint ourselves amount till balance recover
        SellUSSDBuyCollateral();
      }
    }

    function BuyUSSDSellCollateral(uint256 amountToBuy) internal {
      CollateralInfo[] memory collateral = IUSSD(USSD).collateralList();
      //uint amountToBuyLeftUSD = amountToBuy * 1e12 * 1e6 / getOwnValuation();
-     uint amountToBuyLeftUSD = amountToBuy * 1e12;
+     uint amountToBuyLeftUSD = amountToBuy;
```