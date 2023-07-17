# Issue H-1: `StableOracleDAI` calculates `getPriceUSD` with inverted base/rate tokens for Chainlink price 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/102 

## Found by 
0xPkhatri, 0xRobocop, 0xyPhilic, Bahurum, Brenzee, J4de, Juntao, Viktor\_Cortess, juancito, nobody2018, pengun, sashik\_eth, shaka, twicek
## Summary

`StableOracleDAI::getPriceUSD()` calculates the average price between the Uniswap pool price for a pair and the Chainlink feed as part of its result.

The problem is that it uses `WETH/DAI` as the base/rate tokens for the pool, and `DAI/ETH` for the Chainlink feed, which is the opposite.

This will incur in a huge price difference that will impact on the amount of USSD tokens being minted, while requesting the price from this oracle.

## Vulnerability Detail

In `StableOracleDAI::getPrice()` the `price` from the Chainlink feed `priceFeedDAIETH` returns the price as DAI/ETH.

This can be checked on [Etherscan](https://etherscan.io/address/0x773616E4d11A78F511299002da57A0a94577F1f4#readContract#F10) and the [Chainlink Feeds Page](https://docs.chain.link/data-feeds/price-feeds/addresses/).

Also note the comment on the code is misleading, as it is refering to another pair:

> chainlink price data is 8 decimals for WETH/USD

```solidity
/// constructor
24:    priceFeedDAIETH = AggregatorV3Interface(
25:        0x773616E4d11A78F511299002da57A0a94577F1f4
26:    );

/// getPrice()
46:    // chainlink price data is 8 decimals for WETH/USD, so multiply by 10 decimals to get 18 decimal fractional
47:    //(uint80 roundID, int256 price, uint256 startedAt, uint256 timeStamp, uint80 answeredInRound) = priceFeedDAIETH.latestRoundData();
48:    (, int256 price, , , ) = priceFeedDAIETH.latestRoundData();
```

[Link to code](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L46-L48)

On the other hand, the price coming from the Uniswap pool `DAIWethPrice` returns the price as `WETH/DAI`.

Note that the relation WETH/DAI is given by the orders of the token addresses passed as arguments, being the first the base token, and the second the quote token.

Also note that the variable name `DAIWethPrice` is misleading as well as the base/rate are the opposite (although this doesn't affect the code).

```solidity
    uint256 DAIWethPrice = DAIEthOracle.quoteSpecificPoolsWithTimePeriod(
        1000000000000000000, // 1 Eth
        0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2, // WETH (base token) // @audit
        0x6B175474E89094C44Da98b954EedeAC495271d0F, // DAI (quote token) // @audit
        pools, // DAI/WETH pool uni v3
        600 // period
    );
```

[Link to code](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L36-L42)

Finally, both values are used to calculate an average price of in `((DAIWethPrice + uint256(price) * 1e10) / 2)`.

But as seen, one has price in `DAI/ETH` and the other one in `WETH/DAI`, which leads to an incorrect result.

```solidity
    return
        (wethPriceUSD * 1e18) /
        ((DAIWethPrice + uint256(price) * 1e10) / 2);
```

[Link to code](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L50C15-L52)

The average will be lower in this case, and the resulting price higher. 

This will be used by `USSD::mintForToken()` for calculating the amount of tokens to mint for the user, and thus giving them much more than they should.

Also worth mentioning that `USSDRebalancer::rebalance()` also relies on the result of this price calculation and will make it perform trades with incorrect values.

## Impact

Users will receive far more USSD tokens than they should when they call `mintForToken()`, ruining the token value.

When performed the `USSDRebalancer::rebalance()`, all the calculations will be broken for the DAI oracle, leading to incorrect pool trades due to the error in `getPrice()`

## Code Snippet

- https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L46C28-L48
- https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L36-L42
- https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L50C15-L52

## Tool used

Manual Review

## Recommendation

Calculate the inverse of the `price` returned by the Chainlink feed so that it can be averaged with the pool price, making sure that both use the correct `WETH/DAI` and `ETH/DAI` base/rate tokens.





## Discussion

**T1MOH593**

Escalate for 10 USDC

This is not a duplicate of https://github.com/sherlock-audit/2023-05-USSD-judging/issues/909.
It tells about using DAI/ETH instead of ETH/DAI on Chainlink. And #909 tells about completely different issue with oracles

**sherlock-admin**

 > Escalate for 10 USDC
> 
> This is not a duplicate of https://github.com/sherlock-audit/2023-05-USSD-judging/issues/909.
> It tells about using DAI/ETH instead of ETH/DAI on Chainlink. And #909 tells about completely different issue with oracles

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**0xJuancito**

Escalate for 10 USDC

Agree with the previous comment. 

This is an **independent High** impact finding. It is not a duplicate of https://github.com/sherlock-audit/2023-05-USSD-judging/issues/909, and hasn't been exposed by other findings selected for report.

It's main point is explained on the Summary:

> The problem is that it uses WETH/DAI as the base/rate tokens for the pool, and DAI/ETH for the Chainlink feed, which is the opposite.

A more detailed explanation and recommendation to fix it is included on the rest of the report.

Possible duplicates:

- https://github.com/sherlock-audit/2023-05-USSD-judging/issues/795
- https://github.com/sherlock-audit/2023-05-USSD-judging/issues/774
- https://github.com/sherlock-audit/2023-05-USSD-judging/issues/491
- https://github.com/sherlock-audit/2023-05-USSD-judging/issues/269

**sherlock-admin**

 > Escalate for 10 USDC
> 
> Agree with the previous comment. 
> 
> This is an **independent High** impact finding. It is not a duplicate of https://github.com/sherlock-audit/2023-05-USSD-judging/issues/909, and hasn't been exposed by other findings selected for report.
> 
> It's main point is explained on the Summary:
> 
> > The problem is that it uses WETH/DAI as the base/rate tokens for the pool, and DAI/ETH for the Chainlink feed, which is the opposite.
> 
> A more detailed explanation and recommendation to fix it is included on the rest of the report.
> 
> Possible duplicates:
> 
> - https://github.com/sherlock-audit/2023-05-USSD-judging/issues/795
> - https://github.com/sherlock-audit/2023-05-USSD-judging/issues/774
> - https://github.com/sherlock-audit/2023-05-USSD-judging/issues/491
> - https://github.com/sherlock-audit/2023-05-USSD-judging/issues/269

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**ctf-sec**

See my comments in https://github.com/sherlock-audit/2023-05-USSD-judging/issues/555

**hrishibhat**

Result:
High
Has duplicates 
This is a valid separate issue.


**sherlock-admin**

Escalations have been resolved successfully!

Escalation status:
- [T1MOH593](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/102/#issuecomment-1604525700): accepted
- [0xJuancito](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/102/#issuecomment-1605034302): accepted

# Issue H-2: `USSDRebalancer.sol#SellUSSDBuyCollateral` the check of whether collateral is DAI is wrong 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/193 

## Found by 
0xHati, 0xlmanini, 0xyPhilic, Angry\_Mustache\_Man, Bahurum, J4de, Vagner, WATCHPUG, ast3ros, auditsea, carrotsmuggler, curiousapple, innertia, nobody2018, qpzm, saidam017, sashik\_eth, simon135, smiling\_heretic, toshii
## Summary

The `SellUSSDBuyCollateral` function use `||` instand of `&&` to check whether the collateral is DAI. It is wrong and may cause `SellUSSDBuyCollateral` function revert.

## Vulnerability Detail

```solidity
196       for (uint256 i = 0; i < collateral.length; i++) {
197         uint256 collateralval = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * 1e18 / (10**IERC20MetadataUpgradeable(collateral[i].token).decimals()) * collateral[i].oracle.getPriceUSD() / 1e18;
198         if (collateralval * 1e18 / ownval < collateral[i].ratios[flutter]) {
199 >>        if (collateral[i].token != uniPool.token0() || collateral[i].token != uniPool.token1()) {
200             // don't touch DAI if it's needed to be bought (it's already bought)
201             IUSSD(USSD).UniV3SwapInput(collateral[i].pathbuy, daibought/portions);
202           }
203         }
204       }
```

Line 199 should use `&&` instand of `||` to ensure that the token is not DAI. If the token is DAI, the `UniV3SwapInput` function will revert because that DAI's `pathbuy` is empty.

## Impact

The `SellUSSDBuyCollateral` will revert and USSD will become unstable.

## Code Snippet

https://github.com/USSDofficial/ussd-contracts/blob/f44c726371f3152634bcf0a3e630802e39dec49c/contracts/USSDRebalancer.sol#L199

## Tool used

Manual Review

## Recommendation

```diff
      for (uint256 i = 0; i < collateral.length; i++) {
        uint256 collateralval = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * 1e18 / (10**IERC20MetadataUpgradeable(collateral[i].token).decimals()) * collateral[i].oracle.getPriceUSD() / 1e18;
        if (collateralval * 1e18 / ownval < collateral[i].ratios[flutter]) {
-         if (collateral[i].token != uniPool.token0() || collateral[i].token != uniPool.token1()) {
+         if (collateral[i].token != uniPool.token0() && collateral[i].token != uniPool.token1()) {
            // don't touch DAI if it's needed to be bought (it's already bought)
            IUSSD(USSD).UniV3SwapInput(collateral[i].pathbuy, daibought/portions);
          }
        }
      }
```

# Issue H-3: The getOwnValuation() function contains errors in the price calculation 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/222 

## Found by 
0xPkhatri, 0xpinky, AlexCzm, Bauer, J4de, carrotsmuggler, kiki\_dev, peanuts, sam\_gmk, sashik\_eth, simon135, theOwl, twicek, warRoom
## Summary
The getOwnValuation() function in the provided code has incorrect price calculation logic when token0() or token1() is equal to USSD. The error leads to inaccurate price calculations.

## Vulnerability Detail
The `USSDRebalancer.getOwnValuation()` function calculates the price based on the sqrtPriceX96 value obtained from the uniPool.slot0() function. The calculation depends on whether token0() is equal to USSD or not.
If token0() is equal to USSD, the price calculation is performed as follows:
```solidity
  price = uint(sqrtPriceX96)*(uint(sqrtPriceX96))/(1e6) >> (96 * 2);
```
However,there is an error in the price calculation logic. The calculation should be:
```solidity
price = uint(sqrtPriceX96) * uint(sqrtPriceX96) * 1e6 >> (96 * 2);

```
If token0() is not equal to USSD, the price calculation is slightly different:
```solidity
 price = uint(sqrtPriceX96)*(uint(sqrtPriceX96))*(1e18 /* 1e12 + 1e6 decimal representation */) >> (96 * 2);
        // flip the fraction
        price = (1e24 / price) / 1e12;
```
The calculation should be:
```solidity
 price = uint(sqrtPriceX96)*(uint(sqrtPriceX96))*(1e6 /* 1e12 + 1e6 decimal representation */) >> (96 * 2);
        // flip the fraction
        price = (1e24 / price) / 1e12;
```
Reference link:
https://blog.uniswap.org/uniswap-v3-math-primer

## Impact
The incorrect price calculation in the getOwnValuation() function can lead to significant impact on the valuation of assets in the UniSwap V3 pool. The inaccurate prices can result in incorrect asset valuations, which may affect trading decisions, liquidity provision, and overall financial calculations based on the UniSwap V3 pool.

## Code Snippet
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L74
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L76
## Tool used

Manual Review

## Recommendation
When token0() is USSD, the correct calculation should be uint(sqrtPriceX96) * uint(sqrtPriceX96) * 1e6 >> (96 * 2).
When token1() is USSD, the correct calculation should be 
```solidity 
price = uint(sqrtPriceX96)*(uint(sqrtPriceX96))*(1e6 /* 1e12 + 1e6 decimal representation */) >> (96 * 2);
        // flip the fraction
        price = (1e24 / price) / 1e12;
```

# Issue H-4: The price from `StableOracleDAI` is returned with the incorrect number of decimals 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/236 

## Found by 
0xStalin, 0xeix, 0xlmanini, Bahurum, Brenzee, Dug, G-Security, PNS, Proxy, SanketKogekar, T1MOH, Vagner, WATCHPUG, ast3ros, ctf\_sec, immeas, juancito, kutugu, n33k, peanuts, pengun, qbs, qpzm, saidam017, sam\_gmk, sashik\_eth, twicek
## Summary

The price returned from the `getPriceUSD` function of the `StableOracleDAI` is scaled up by `1e10`, which results in 28 decimals instead of the intended 18.

## Vulnerability Detail

In `StableOracleDAI` the `getPriceUSD` function is defined as follows...

```solidity
    function getPriceUSD() external view override returns (uint256) {
        address[] memory pools = new address[](1);
        pools[0] = 0x60594a405d53811d3BC4766596EFD80fd545A270;
        uint256 DAIWethPrice = DAIEthOracle.quoteSpecificPoolsWithTimePeriod(
            1000000000000000000, // 1 Eth
            0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2, // WETH (base token)
            0x6B175474E89094C44Da98b954EedeAC495271d0F, // DAI (quote token)
            pools, // DAI/WETH pool uni v3
            600 // period
        );

        uint256 wethPriceUSD = ethOracle.getPriceUSD();

        // chainlink price data is 8 decimals for WETH/USD, so multiply by 10 decimals to get 18 decimal fractional
        //(uint80 roundID, int256 price, uint256 startedAt, uint256 timeStamp, uint80 answeredInRound) = priceFeedDAIETH.latestRoundData();
        (, int256 price,,,) = priceFeedDAIETH.latestRoundData();

        return (wethPriceUSD * 1e18) / ((DAIWethPrice + uint256(price) * 1e10) / 2);
    }
```

The assumption is made that the `DAIWethPrice` is 8 decimals, and is therefore multiplied by `1e10` in the return statement to scale it up to 18 decimals. 

The _other_ price feeds used in the protocol are indeed received with decimals, however, the Chainlink DAI/ETH price feed returns a value with 18 decimals as can be seen on their site.

https://docs.chain.link/data-feeds/price-feeds/addresses

## Impact

This means that the price returned from the `getPriceUSD` function is scaled up by `1e10`, which results in 28 decimals instead of the intended 18, drastically overvaluing the DAI/USD price.

This will result in the USSD token price being a tiny fraction of what it is intended to be. Instead of being pegged to $1, it will be pegged to $0.0000000001, completely defeating the purpose of the protocol.

For example, if a user calls `USSD.mintForToken`, supplying DAI, they'll be able to mint `1e10` times more USSD than intended.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L33-L53

## Tool used

Manual Review

## Recommendation

Remove the `* 1e10` from the return statement.

```diff
-   return (wethPriceUSD * 1e18) / ((DAIWethPrice + uint256(price) * 1e10) / 2);
+   return (wethPriceUSD * 1e18) / (DAIWethPrice + uint256(price) / 2);
```

# Issue H-5: Price calculation susceptible to flashloan exploits 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/451 

## Found by 
0xHati, Bauchibred, Bauer, BugBusters, Fanz, JohnnyTime, Kodyvim, Schpiel, SensoYard, VAD37, WATCHPUG, \_\_141345\_\_, blockdev, carrotsmuggler, chalex.eth, coincoin, immeas, juancito, kiki\_dev, n33k, ni8mare, nobody2018, peanuts, qbs, qckhp, shaka, shogoki, simon135, smiling\_heretic, tallo, theOwl, tsvetanovv
## Summary

Contract uses uniswap `slot0` price instead of TWAP price. `slot0` price can be manipulated with flash loans.

## Vulnerability Detail

The contract uses the uniswap DAI-USSD pool to get the price of USSD. It however uses the instantaneous price from `slot0` instead of the TWAP price. The `slot0` price is calculated from the ratios of the assets. This ratio can however be manipulated by buying/selling assets in the pool.

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L71-L80

Thus any user can take a flashloan, use those funds to manipulate the price of USSD, and then trigger a rebalance. The attacks can be made profitable by providing just-in-time liquidity to the various pools that `reabalance` interacts with, draining the contract of collateral through arbitrage.

## Impact

Price can be manipulated and `rebalance` can be called any time. Susceptible to flash loan exploits.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L71-L80

## Tool used

Manual Review

## Recommendation

Use TWAP price instead of `slot0` price. [Here](https://github.com/charmfinance/alpha-vaults-contracts/blob/07db2b213315eea8182427be4ea51219003b8c1a/contracts/AlphaStrategy.sol#L136-L144) is an example implementation of TWAP.

# Issue H-6: Wrong computation of the amountToSellUnit variable 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/535 

## Found by 
0xRobocop, 0xlmanini, Aymen0909, Bahurum, Bauer, Juntao, Nyx, Proxy, VAD37, Vagner, WATCHPUG, \_\_141345\_\_, auditsea, carrotsmuggler, immeas, innertia, kiki\_dev, pengun, qpzm, saidam017, sakshamguruji, toshii, tvdung94
## Summary

The variable `amountToSellUnits` is computed wrongly in the code which will lead to an incorrect amount of collateral to be sold.

## Vulnerability Detail

The `BuyUSSDSellCollateral()` function is used to sell collateral during a peg-down recovery event. The computation of the amount to sell is computed using the following formula:

```solidity
// @audit-issue Wrong computation
uint256 amountToSellUnits = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * ((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18) / 1e18;
```

The idea is to sell an amount which is equivalent (in USD) to the ratio of `amountToBuyLeftUSD / collateralval`. Flattening the equation it ends up as:

```solidity
uint256 amountToSellUnits = (collateralBalance * amountToBuyLeftUSD * 1e18) / (collateralval * 1e18 * 1e18);

// Reducing the equation
uint256 amountToSellUnits = (collateralBalance * amountToBuyLeftUSD) / (collateralval * 1e18);
```

`amountToBuyLeftUSD` and `collateralval` already have 18 decimals so their decimals get cancelled together which will lead the last 1e18 factor as not necessary.

## Impact

The contract will sell an incorrect amount of collateral during a peg-down recovery event.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/6d7a9fdfb1f1ed838632c25b6e1b01748d0bafda/ussd-contracts/contracts/USSDRebalancer.sol#L121

## Tool used

Manual Review

## Recommendation

Delete the last 1e18 factor

# Issue H-7: Not using slippage parameter or deadline while swapping on UniswapV3 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/673 

## Found by 
0xPkhatri, 0xRan4212, 0xRobocop, 0xSmartContract, 0xStalin, 0xeix, 0xpinky, 0xyPhilic, Angry\_Mustache\_Man, Auditwolf, Bahurum, Bauchibred, Bauer, BlockChomper, Brenzee, BugBusters, BugHunter101, CodeFoxInc, Delvir0, Dug, Fanz, HonorLt, J4de, JohnnyTime, Juntao, Kodyvim, Kose, Lilyjjo, Madalad, MohammedRizwan, Nyx, PokemonAuditSimulator, Proxy, RaymondFam, Saeedalipoor01988, Schpiel, SensoYard, T1MOH, TheNaubit, Tricko, Viktor\_Cortess, WATCHPUG, \_\_141345\_\_, anthony, ast3ros, berlin-101, blackhole, blockdev, carrotsmuggler, chaithanya\_gali, chalex.eth, coincoin, ctf\_sec, curiousapple, dacian, evilakela, eyexploit, immeas, innertia, jah, jprod15, juancito, kie, kiki\_dev, kutugu, lil.eth, m4ttm, martin, n33k, ni8mare, nobody2018, peanuts, qbs, qckhp, qpzm, saidam017, sakshamguruji, sam\_gmk, sashik\_eth, shaka, shealtielanz, shogoki, simon135, slightscan, tallo, theOwl, toshii, twicek, warRoom
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

# Issue H-8: Lack of access control for `mintRebalancer()` and `burnRebalancer()` 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/777 

## Found by 
0x2e, 0xAzez, 0xHati, 0xMojito, 0xPkhatri, 0xRobocop, 0xSmartContract, 0xStalin, 0xeix, 0xyPhilic, 14si2o\_Flint, AlexCzm, Angry\_Mustache\_Man, Aymen0909, Bahurum, Bauchibred, Bauer, BlockChomper, Brenzee, BugBusters, BugHunter101, Delvir0, DevABDee, Dug, Fanz, GimelSec, HonorLt, J4de, JohnnyTime, Juntao, Kodyvim, Kose, Lilyjjo, Madalad, Nyx, PokemonAuditSimulator, RaymondFam, Saeedalipoor01988, SanketKogekar, Schpiel, SensoYard, T1MOH, TheNaubit, Tricko, VAD37, Vagner, WATCHPUG, \_\_141345\_\_, anthony, ast3ros, auditsea, berlin-101, blackhole, blockdev, carrotsmuggler, chainNue, chalex.eth, cjm00n, coincoin, coryli, ctf\_sec, curiousapple, dacian, evilakela, georgits, giovannidisiena, immeas, innertia, jah, juancito, kie, kiki\_dev, lil.eth, m4ttm, mahdikarimi, mrpathfindr, n33k, neumo, ni8mare, nobody2018, pavankv241, pengun, qbs, qckhp, qpzm, ravikiran.web3, saidam017, sam\_gmk, sashik\_eth, shaka, shealtielanz, shogoki, simon135, slightscan, smiling\_heretic, tallo, theOwl, the\_endless\_sea, toshii, tsvetanovv, tvdung94, twcctop, twicek, vagrant, ver0759, warRoom, whiteh4t9527, ww4tson, yy
## Summary

Lack of access control in `USSD.mintRebalancer()` and `USSD.burnRebalancer()` can lead to a denial-of-service attack and malfunction of the rebalancer as it can alter `totalSupply`, which is used in `rebalancer.SellUSSDBuyCollateral` to calculate `ownval`.

## Vulnerability Detail

Based on the context, `USSD.mintRebalancer()` should be `onlyBalancer` as it should only be allowed to be called by the rebalancer.

However, both `USSD.mintRebalancer()` and `USSD.burnRebalancer()` lack access control in the current implementation.

## Impact

An attacker can mint an amount of `type(uint256).max - totalSupply()` and cause a denial-of-service attack by preventing anyone else from minting.

Additionally, minting will also change the `totalSupply` which alters the `collateralFactor` and cause the rebalancer to malfunction, as the `SellUSSDBuyCollateral()` function relies on the `USSD.collateralFactor()`.

The `totalSupply` is also used in `rebalancer.SellUSSDBuyCollateral` to calculate the `ownval`.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L204-L210

```solidity
function mintRebalancer(uint256 amount) public override {
    _mint(address(this), amount);
}

function burnRebalancer(uint256 amount) public override {
    _burn(address(this), amount);
}
```

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L92-L107

```solidity
    function rebalance() override public {
      uint256 ownval = getOwnValuation();
      (uint256 USSDamount, uint256 DAIamount) = getSupplyProportion();
      if (ownval < 1e6 - threshold) {
        // peg-down recovery
        BuyUSSDSellCollateral((USSDamount - DAIamount / 1e12)/2);
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
```

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L179-L194

```solidity
function collateralFactor() public view override returns (uint256) {
    uint256 totalAssetsUSD = 0;
    for (uint256 i = 0; i < collateral.length; i++) {
        totalAssetsUSD +=
            (((IERC20Upgradeable(collateral[i].token).balanceOf(
                address(this)
            ) * 1e18) /
                (10 **
                    IERC20MetadataUpgradeable(collateral[i].token)
                        .decimals())) *
                collateral[i].oracle.getPriceUSD()) /
            1e18;
    }

    return (totalAssetsUSD * 1e6) / totalSupply();
}
```

## Tool used

Manual Review

## Recommendation

`USSD.mintRebalancer()` should be `onlyBalancer`.

# Issue H-9: Uniswap v3 pool token balance proportion does not necessarily correspond to the price, and it is easy to manipulate. 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/808 

## Found by 
0xRan4212, Bahurum, VAD37, WATCHPUG, curiousapple, mahdikarimi, n33k, nobody2018, simon135
## Summary

`getSupplyProportion()` retrieves Uniswap v3 pool balances, but the proportion of pool tokens doesn't always correspond to the price. If `ownval` is less than `1e6 - threshold`, `USSDamount` may be lower than `DAIamount`, causing L97 `USSDamount - DAIamount` to revert due to underflow. Proportion can be easily manipulated, which can be exploited by attackers.

## Vulnerability Detail

`getSupplyProportion()` retrieves the balances of the Uniswap v3 pool. However, due to the different designs of Uniswap v3 and Uniswap v2, the proportion of pool tokens does not necessarily correspond to the price.

As a result, if `ownval` is less than `1e6 - threshold` (e.g. 0.95), `USSDamount` may be lower than `DAIamount`, causing L97 `USSDamount - DAIamount` to revert due to underflow.

Additionally, the pool contract holds accumulative fees on its balances, which are not impacted by price changes.

---

Furthermore, the proportion can be easily manipulated with minimal cost, which can be exploited by attackers.

If the price of USSD goes over-peg (which can happen naturally), an attacker can take advantage by following these steps:

1. Add single leg liquidity of DAI to the DAI/USSD pool at an exorbitantly high price range, such as 1 DAI == 1000-2000 USSD.
2. Manipulate the price of the collateral asset, such as WETH, to a higher price.
3. Place a limit-order like JIT liquidity at a higher price in the WETH/DAI pool.
4. Trigger `rebalance() -> SellUSSDBuyCollateral()`, but it will mint and sell much more than expected (the amount needed to bring the peg back) as the `DAIamount` is significantly higher than `USSDamount` at a manipulated high price. This will buy the limit order from step 3.
5. Reverse the price of WETH/DAI and remove the liquidity placed at step 1.

## Impact

1. `rebalance()` may malfunction if the proportion is not as expected.
2. Manipulating the proportion can result in the protocol selling a significantly larger amount of collateral assets than intended. If the collateral asset is also manipulated, it would be sold at a manipulated price, causing even larger damage to the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L13-L16

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L83-L107

## Tool used

Manual Review

## Recommendation

Instead of using the pool balances to calculate the delta amount required to restore the peg, a more complex formula that considers the liquidity range should be used.



## Discussion

**0xJuancito**

Escalate for 10 USDC

This issue is already addressed on #451 and its duplicates

All of them refer to manipulation of Uniswap v3 pool and calling `rebalance()` to manipulate USSD price. Other duplicate findings address the same issue as here with `getSupplyProportion ()` as well, like https://github.com/sherlock-audit/2023-05-USSD-judging/issues/92, https://github.com/sherlock-audit/2023-05-USSD-judging/issues/731, https://github.com/sherlock-audit/2023-05-USSD-judging/issues/733 just to give some examples.

**sherlock-admin**

 > Escalate for 10 USDC
> 
> This issue is already addressed on #451 and its duplicates
> 
> All of them refer to manipulation of Uniswap v3 pool and calling `rebalance()` to manipulate USSD price. Other duplicate findings address the same issue as here with `getSupplyProportion ()` as well, like https://github.com/sherlock-audit/2023-05-USSD-judging/issues/92, https://github.com/sherlock-audit/2023-05-USSD-judging/issues/731, https://github.com/sherlock-audit/2023-05-USSD-judging/issues/733 just to give some examples.

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**bahurum**

Escalate for 10 USDC

This isssue  is not a duplicate of #451 as claimed by @0xJuancito.

The problem here is not the usage of `slot0()` but calculating the price of a Uniswap V3 pool as ratio of pool reserves, which is fundamentally wrong.
Also, while this issue allows an attack vector which involves manipulation of the price of the pool, the issue already exists without manipulation as the rebalancing will be completely off or would not work at all, and this for any normal non-manipulated pool.


**sherlock-admin**

 > Escalate for 10 USDC
> 
> This isssue  is not a duplicate of #451 as claimed by @0xJuancito.
> 
> The problem here is not the usage of `slot0()` but calculating the price of a Uniswap V3 pool as ratio of pool reserves, which is fundamentally wrong.
> Also, while this issue allows an attack vector which involves manipulation of the price of the pool, the issue already exists without manipulation as the rebalancing will be completely off or would not work at all, and this for any normal non-manipulated pool.
> 

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**0xJuancito**

Escalate for 10 USDC

For clarification from my previous comment, I'm pointing that many issues have been judged as a duplicate of #451 as stated on my comment due to this root cause as well, not necessarily with `slot0`, but the current use of the pool.

> This issue is already addressed on https://github.com/sherlock-audit/2023-05-USSD-judging/issues/451 and its duplicates

Just to give some examples:

- https://github.com/sherlock-audit/2023-05-USSD-judging/issues/731

> getSupplyProportion uses Uniswap V3 pool tokens balances which are easily manipulated. The protocol rebalances the USSD/DAI token proportion to 50/50 to rebalance the USSD/DAI price. This works for Uniswap V2 pools but does not work for Uniswap V3 pools.

- https://github.com/sherlock-audit/2023-05-USSD-judging/issues/733

> The getSupplyProportion() function, using the balanceOf() function, is designed to maintain the balance of the USSD/DAI pool in order to stabilize the USSD value at $1.

> However, this balance can be manipulated, particularly through UniswapPool flashloan, which facilitates the alteration of the balanceOf() value of both USSD and DAI in the pool. This then tricks the USSDRebalancer.rebalance() function into swapping half the total pool value.

My suggestion is to keep the consistency in the judging and group them all, as similar issues have been already grouped under #451.

**sherlock-admin**

 > Escalate for 10 USDC
> 
> For clarification from my previous comment, I'm pointing that many issues have been judged as a duplicate of #451 as stated on my comment due to this root cause as well, not necessarily with `slot0`, but the current use of the pool.
> 
> > This issue is already addressed on https://github.com/sherlock-audit/2023-05-USSD-judging/issues/451 and its duplicates
> 
> Just to give some examples:
> 
> - https://github.com/sherlock-audit/2023-05-USSD-judging/issues/731
> 
> > getSupplyProportion uses Uniswap V3 pool tokens balances which are easily manipulated. The protocol rebalances the USSD/DAI token proportion to 50/50 to rebalance the USSD/DAI price. This works for Uniswap V2 pools but does not work for Uniswap V3 pools.
> 
> - https://github.com/sherlock-audit/2023-05-USSD-judging/issues/733
> 
> > The getSupplyProportion() function, using the balanceOf() function, is designed to maintain the balance of the USSD/DAI pool in order to stabilize the USSD value at $1.
> 
> > However, this balance can be manipulated, particularly through UniswapPool flashloan, which facilitates the alteration of the balanceOf() value of both USSD and DAI in the pool. This then tricks the USSDRebalancer.rebalance() function into swapping half the total pool value.
> 
> My suggestion is to keep the consistency in the judging and group them all, as similar issues have been already grouped under #451.

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**ctf-sec**

Consider this not a duplicate of #451 

and will redo some duplicate later

**0xRan4212**

Escalate for 10 USDC

https://github.com/sherlock-audit/2023-05-USSD-judging/issues/931 is a dup of this issue.

**hrishibhat**

Result:
High
Has duplicates
This issue is not a duplicate of #451, some of the duplications are changed accordingly. 

**sherlock-admin**

Escalations have been resolved successfully!

Escalation status:
- [0xJuancito](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/808/#issuecomment-1605929297): accepted
- [bahurum](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/808/#issuecomment-1605947205): accepted
- [0xJuancito](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/808/#issuecomment-1606170316): accepted

# Issue H-10: Wrong Oracle feed addresses 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/817 

## Found by 
0xGusMcCrae, 0xHati, 0xPkhatri, 0xRobocop, 0xStalin, 0xeix, 0xlmanini, 0xyPhilic, 14si2o\_Flint, ADM, Aymen0909, Bahurum, Bauchibred, Bauer, BenRai, Brenzee, BugHunter101, Delvir0, DevABDee, Dug, G-Security, GimelSec, HonorLt, J4de, JohnnyTime, Juntao, Kirkeelee, Kodyvim, Kose, Lilyjjo, Madalad, PNS, PTolev, PokemonAuditSimulator, Proxy, Saeedalipoor01988, SaharDevep, Schpiel, SensoYard, T1MOH, TheNaubit, Vagner, Viktor\_Cortess, WATCHPUG, \_\_141345\_\_, ashirleyshe, ast3ros, berlin-101, blockdev, chainNue, chalex.eth, ck, ctf\_sec, curiousapple, dacian, evilakela, giovannidisiena, immeas, innertia, juancito, kie, kiki\_dev, kutugu, lil.eth, martin, mrpathfindr, neumo, ni8mare, nobody2018, peanuts, pengun, qpzm, ravikiran.web3, saidam017, sakshamguruji, sam\_gmk, sashik\_eth, shaka, shogoki, simon135, theOwl, the\_endless\_sea, toshii, twicek, ustas, whiteh4t9527
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

# Issue H-11: Oracle price should be denominated in DAI instead of USD 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/909 

## Found by 
T1MOH, WATCHPUG
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



## Discussion

**0xJuancito**

Escalate for 10 USDC

The report assumes that the attacker would be able to sell 1100 USSD for 1100 DAI:

> 2. Sell 1100 USSD for 1100 DAI, driving the USSD to a lower price against DAI, say 1 USSD to 0.98 DAI.

This wouldn't be possible, as the USSD/DAI pool would be arbitraged and the attacker will only be losing money on each mint. It also assumes manipulating the price, which is already mentioned in https://github.com/sherlock-audit/2023-05-USSD-judging/issues/451.

This can be considered either informational or a duplicate

**sherlock-admin**

 > Escalate for 10 USDC
> 
> The report assumes that the attacker would be able to sell 1100 USSD for 1100 DAI:
> 
> > 2. Sell 1100 USSD for 1100 DAI, driving the USSD to a lower price against DAI, say 1 USSD to 0.98 DAI.
> 
> This wouldn't be possible, as the USSD/DAI pool would be arbitraged and the attacker will only be losing money on each mint. It also assumes manipulating the price, which is already mentioned in https://github.com/sherlock-audit/2023-05-USSD-judging/issues/451.
> 
> This can be considered either informational or a duplicate

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**ctf-sec**

Emm do not see why this is not possible

> Sell 1100 USSD for 1100 DAI, driving the USSD to a lower price against DAI, say 1 USSD to 0.98 DAI.

the attack path described is still valid in the original report

**hrishibhat**

Result:
High
Has duplicates
The attack is possible and the rebalance is design to create maintain the 1:1 ratio between USSD to DAI.

**sherlock-admin**

Escalations have been resolved successfully!

Escalation status:
- [0xJuancito](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/909/#issuecomment-1605924088): rejected

# Issue M-1: Calls to Oracles don't check for stale prices 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/31 

## Found by 
0x2e, 0xHati, 0xPkhatri, 0xRobocop, 0xSmartContract, 0xStalin, 0xeix, 0xlmanini, 0xyPhilic, Angry\_Mustache\_Man, Aymen0909, Bauchibred, Bauer, Brenzee, BugBusters, Delvir0, DevABDee, Diana, Dug, Fanz, GimelSec, HonorLt, J4de, Kodyvim, Kose, Lilyjjo, Madalad, MohammedRizwan, Nyx, PNS, PTolev, Pheonix, PokemonAuditSimulator, Proxy, RaymondFam, Saeedalipoor01988, SaharDevep, SanketKogekar, Schpiel, T1MOH, TheNaubit, VAD37, WATCHPUG, \_\_141345\_\_, ast3ros, berlin-101, capy\_, chainNue, chaithanya\_gali, chalex.eth, ctf\_sec, curiousapple, dacian, evilakela, georgits, giovannidisiena, immeas, josephdara, juancito, kiki\_dev, kutugu, lil.eth, martin, ni8mare, nobody2018, pavankv241, peanuts, qbs, qckhp, saidam017, sakshamguruji, sam\_gmk, sashik\_eth, sayan\_, shaka, shealtielanz, simon135, ss3434, tallo, theOwl, toshii, tsvetanovv, twicek, ustas, vagrant, w42d3n, warRoom, whiteh4t9527
## Summary
Calls to Oracles don't check for stale prices.

## Vulnerability Detail
None of the oracle calls check for stale prices, for example [StableOracleDAI.getPriceUSD()](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L48):
```solidity
(, int256 price, , , ) = priceFeedDAIETH.latestRoundData();

return
    (wethPriceUSD * 1e18) /
    ((DAIWethPrice + uint256(price) * 1e10) / 2);
```

## Impact
Oracle price feeds can become stale due to a variety of [reasons](https://ethereum.stackexchange.com/questions/133242/how-future-resilient-is-a-chainlink-price-feed/133843#133843). Using a stale price will result in incorrect calculations in most of the key functionality of USSD & USSDRebalancer contracts.

## Code Snippet
[StableOracleDAI.getPriceUSD()](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L48)
[StableOracleWBGL.getPriceUSD()](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBGL.sol#L36-L38)
[StableOracleWBTC.getPriceUSD()](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L23-L25)
[StableOracleWETH.getPriceUSD()](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWETH.sol#L23-L25)

## Tool used
Manual Review

## Recommendation
Read the ``updatedAt`` parameter from the calls to ``latestRoundData()`` and verify that it isn't older than a set amount, eg:

```solidity
if (updatedAt < block.timestamp - 60 * 60 /* 1 hour */) {
   revert("stale price feed");
}
```

# Issue M-2: Because of missing slippage parameter, mintForToken() can be front-runned 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/97 

## Found by 
0xRobocop, Aymen0909, GimelSec, Kose, cryptostellar5, qbs, shealtielanz
## Summary
Missing slippage parameter in ```mintForToken()``` makes it vulnerable to front-run attacks and exposes users to unwanted slippage.
## Vulnerability Detail
The current implementation of the ```mintForToken()``` function lacks a parameter for controlling slippage, which makes it vulnerable to front-run attacks. Transactions involving large volumes are particularly at risk, as the minting process can be manipulated, resulting in price impact. This manipulation allows the reserves of the pool to be controlled, enabling a frontrunner to make the transferred token to appear more valuable than its actual worth. Consequently, when users mint USSD, they may receive USSD that are worth significantly less than the value of their real worth. This lack of slippage control resembles a swap without a limit on value manipulation.

## Impact
User will be vulnerable to front-run attacks and receive less USSD from their expectation.
## Code Snippet
[USSD.sol#L150-L167](https://github.com/USSDofficial/ussd-contracts/blob/f44c726371f3152634bcf0a3e630802e39dec49c/contracts/USSD.sol#L150-L167)
```solidity
/// Mint specific AMOUNT OF STABLE by giving token
    function mintForToken(
        address token,
        uint256 tokenAmount,
        address to
    ) public returns (uint256 stableCoinAmount) {
        require(hasCollateralMint(token), "unsupported token");

        IERC20Upgradeable(token).safeTransferFrom(
            msg.sender,
            address(this),
            tokenAmount
        );
        stableCoinAmount = calculateMint(token, tokenAmount);
        _mint(to, stableCoinAmount);

        emit Mint(msg.sender, to, token, tokenAmount, stableCoinAmount);
    }
```
## Tool used

Manual Review

## Recommendation
Consider adding a ```minAmountOut``` parameter.



## Discussion

**sherlock-admin**

> Removed: comment left for traceability 
> I think this is not a valid medium. 
> As the mintForToken function uses oracle prices, which return a weighted average price, it should not be that easy manipulated by Frontrunning.

    You've deleted an escalation for this issue.

**kosedogus**

> Escalate for 10USDC I think this is not a valid medium. As the mintForToken function uses oracle prices, which return a weighted average price, it should not be that easy manipulated by Frontrunning.

Weighted average prices does not guarantee that trades (minting in this case)  executed based on the average price will be free from slippage. TWAP's are obviously vulnerable to slippage attacks. I also would like to remind that there is another valid issue due to the lack of slippage parameter in uniRouter. (That function also uses TWAP) 

**Shogoki**

Okay, I think you are right.
I removed the escalation 

# Issue M-3: rebalance process incase of  selling the collateral, could revert because of underflow calculation 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/111 

## Found by 
0xHati, Dug, GimelSec, Juntao, PokemonAuditSimulator, T1MOH, WATCHPUG, XDZIBEC, ast3ros, saidam017, toshii, tsvetanovv, twicek
## Summary

rebalance process, will try to sell the collateral in case of peg-down. However, the process can revert because the calculation can underflow.

## Vulnerability Detail

Inside `rebalance()` call, if `BuyUSSDSellCollateral()` is triggered, it will try to sell the current collateral to `baseAsset`. The asset that will be sold (`amountToSellUnits`) first calculated. Then swap it to `baseAsset` via uniswap. However, when subtracting `amountToBuyLeftUSD`, it with result of `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)`. There is no guarantee `amountToBuyLeftUSD` always bigger than `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)`.

This causing the call could revert in case `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)` > `amountToBuyLeftUSD`.

There are two branch where `amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)` is performed : 

1. Incase `collateralval > amountToBuyLeftUSD`

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L116-L125

`collateralval` is calculated using oracle price, thus the result of swap not guaranteed to reflect the proportion of `amountToBuyLefUSD` against `collateralval` ratio, and could result in returning `baseAsset` larger than expected. And potentially  `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)` > `amountToBuyLeftUSD`

```solidity
        uint256 collateralval = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * 1e18 / (10**IERC20MetadataUpgradeable(collateral[i].token).decimals()) * collateral[i].oracle.getPriceUSD() / 1e18;
        if (collateralval > amountToBuyLeftUSD) {
          // sell a portion of collateral and exit
          if (collateral[i].pathsell.length > 0) {
            uint256 amountBefore = IERC20Upgradeable(baseAsset).balanceOf(USSD);
            uint256 amountToSellUnits = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * ((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18) / 1e18;
            IUSSD(USSD).UniV3SwapInput(collateral[i].pathsell, amountToSellUnits);
            amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
            DAItosell += (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
          } else {
```

2. Incase `collateralval < amountToBuyLeftUSD`

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L132-L138

This also can't guarantee `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)` < `amountToBuyLeftUSD`.

```solidity
          if (collateralval >= amountToBuyLeftUSD / 20) {
            uint256 amountBefore = IERC20Upgradeable(baseAsset).balanceOf(USSD);
            // sell all collateral and move to next one
            IUSSD(USSD).UniV3SwapInput(collateral[i].pathsell, IERC20Upgradeable(collateral[i].token).balanceOf(USSD));
            amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
            DAItosell += (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
          }
```

## Impact

Rebalance process can revert caused by underflow calculation.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L116-L125
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L132-L138

## Tool used

Manual Review

## Recommendation

Check if `(IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore)` > `amountToBuyLeftUSD`, in that case, just set `amountToBuyLeftUSD` to 0.

```solidity
          ...
            uint baseAssetChange = IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
            if (baseAssetChange > amountToBuyLeftUSD) {
                amountToBuyLeftUSD = 0;
            } else {
                amountToBuyLeftUSD -= baseAssetChange;
           }
            DAItosell += baseAssetChange;
          ...
```




## Discussion

**hrishibhat**

This is a valid medium

# Issue M-4: StableOracleWBTC use BTC/USD chainlink oracle to price WBTC which is problematic if WBTC depegs 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/310 

## Found by 
Bahurum, Bauchibred, BenRai, RaymondFam, Schpiel, T1MOH, \_\_141345\_\_, chainNue, chalex.eth, kiki\_dev, sashik\_eth


## Summary

The StableOracleWBTC contract utilizes a BTC/USD Chainlink oracle to determine the price of WBTC. However, this approach can lead to potential issues if WBTC were to depeg from BTC. In such a scenario, WBTC would no longer maintain an equivalent value to BTC. This can result in significant problems, including borrowing against a devalued asset and the accumulation of bad debt. Given that the protocol continues to value WBTC based on BTC/USD, the issuance of bad loans would persist, exacerbating the overall level of bad debt.

Important to note that this is like a 2 in 1 report as the same idea could work on the StableOracleWBGL contract too.

## Vulnerability Detail

The vulnerability lies in the reliance on a single BTC/USD Chainlink oracle to obtain the price of WBTC. If the bridge connecting WBTC to BTC becomes compromised and WBTC depegs, WBTC may depeg from BTC. Consequently, WBTC's value would no longer be equivalent to BTC, potentially rendering it worthless (hopefully this never happens). The use of the BTC/USD oracle to price WBTC poses risks to the protocol and its users.

The following code snippet represents the relevant section of the StableOracleWBTC contract responsible for retrieving the price of WBTC using the BTC/USD Chainlink oracle:

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
        // chainlink price data is 8 decimals for WBTC/USD
        return uint256(price) * 1e10;
    }
}
```

NB: key to note that the above pricefeed is set to the wrong aggregator, the correct one is this: `0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599`

## Impact

Should the WBTC bridge become compromised or WBTC depeg from BTC, the protocol would face severe consequences. The protocol would be burdened with a substantial amount of bad debt stemming from outstanding loans secured by WBTC. Additionally, due to the protocol's reliance on the BTC/USD oracle, the issuance of loans against WBTC would persist even if its value has significantly deteriorated. This would lead to an escalation in bad debt, negatively impacting the protocol's financial stability and overall performance.

## Code Snippet

[StableOracleWBTC.sol#L12-L26](https://github.com/sherlock-audit/2023-05-USSD/blob/6d7a9fdfb1f1ed838632c25b6e1b01748d0bafda/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L12-L26)

## Tool used

Manual review

## Recommendation

To mitigate the vulnerability mentioned above, it is strongly recommended to implement a double oracle setup for WBTC pricing. This setup would involve integrating both the BTC/USD Chainlink oracle and an additional on-chain liquidity-based oracle, such as UniV3 TWAP.

The double oracle setup serves two primary purposes. Firstly, it reduces the risk of price manipulation by relying on the Chainlink oracle, which ensures accurate pricing for WBTC. Secondly, incorporating an on-chain liquidity-based oracle acts as a safeguard against WBTC depegging. By monitoring the price derived from the liquidity-based oracle and comparing it to the Chainlink oracle's price, borrowing activities can be halted if the threshold deviation (e.g., 2% lower) is breached.

Adopting a double oracle setup enhances the protocol's stability and minimizes the risks associated with WBTC depegging. It ensures accurate valuation, reduces the accumulation of bad debt, and safeguards the protocol and its users




## Discussion

**Bauchibred**

Escalate for 10 USDC



I believe this issue has been incorrectly duplicated to #817. While I acknowledge the large number of issues submitted during the contest (approximately 1000), it's crucial to clarify that the concern here is not solely related to oracle addresses, despite the inclusion of wrong aggregators and inactive oracle addresses in the report. The main issue at hand is the potential depegging of WBTC, which is a bridged asset.

To address this vulnerability, the recommendation proposes implementing a double oracle setup for WBTC pricing, which serves as a safeguard against WBTC depegging.

To support this escalation, I have provided references to relevant cases:
- #836 is a sponsor-validated issue that emphasizes the importance of not relying solely on 100% of an asset's oracle price. While it may not directly relate to this specific issue, it underscores the need to consider implementing a "deviation threshold" when determining asset prices, particularly in the context of bridged assets.
-  #862, which is a valid duplicate, explores the potential impact of depegging on the protocol within a different context for this contest.
- Additionally, references [1](https://github.com/sherlock-audit/2023-04-jojo-judging/blob/533cfb7357175fc97d6816586e95ad39e07892a7/060-M/320.md) and [2](https://github.com/sherlock-audit/2023-04-jojo-judging/blob/533cfb7357175fc97d6816586e95ad39e07892a7/060-M/104-best.md?plain=1#L5) are previous validated findings from other contests that further emphasize the standalone nature and significance of this issue.

I believe these references provide additional insights into the importance of considering measures to mitigate risks associated with bridged assets and emphasize why this issue should be treated as a standalone concern.



**sherlock-admin**

 > Escalate for 10 USDC
> 
> 
> 
> I believe this issue has been incorrectly duplicated to #817. While I acknowledge the large number of issues submitted during the contest (approximately 1000), it's crucial to clarify that the concern here is not solely related to oracle addresses, despite the inclusion of wrong aggregators and inactive oracle addresses in the report. The main issue at hand is the potential depegging of WBTC, which is a bridged asset.
> 
> To address this vulnerability, the recommendation proposes implementing a double oracle setup for WBTC pricing, which serves as a safeguard against WBTC depegging.
> 
> To support this escalation, I have provided references to relevant cases:
> - #836 is a sponsor-validated issue that emphasizes the importance of not relying solely on 100% of an asset's oracle price. While it may not directly relate to this specific issue, it underscores the need to consider implementing a "deviation threshold" when determining asset prices, particularly in the context of bridged assets.
> -  #862, which is a valid duplicate, explores the potential impact of depegging on the protocol within a different context for this contest.
> - Additionally, references [1](https://github.com/sherlock-audit/2023-04-jojo-judging/blob/533cfb7357175fc97d6816586e95ad39e07892a7/060-M/320.md) and [2](https://github.com/sherlock-audit/2023-04-jojo-judging/blob/533cfb7357175fc97d6816586e95ad39e07892a7/060-M/104-best.md?plain=1#L5) are previous validated findings from other contests that further emphasize the standalone nature and significance of this issue.
> 
> I believe these references provide additional insights into the importance of considering measures to mitigate risks associated with bridged assets and emphasize why this issue should be treated as a standalone concern.
> 
> 

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**twicek**

Escalate for 10 USDC

Comments from [Bauchibred](https://github.com/Bauchibred) regarding the fact that the present issue #310 is not a duplicate of #817 are correct.
However, the main argument of this report is that:
>WBTC may depeg from BTC

The only fact that WBTC/BTC would depeg is questionable. Since it is issued by a centralized entity (BitGo) it should be treated as trusted, because the only way for it to depeg would be via an error of this centralized entity or if the DAO voted a malicious proposal. See for details: https://www.gemini.com/cryptopedia/wbtc-what-is-wrapped-bitcoin#section-how-w-btc-works

Additionally, one of the justifications used to support the escalation regarding the deviation threshold mentioned in [#836 ](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/836). There is no guarantee that the TWAP will not stay within the deviation threshold even in the (very) unlikely event that WBTC/BTC depegs if the depegging happens slowly.

Regarding #862, this is not a duplicate to this report because DAI principal depeg risk comes from depegging of the underlying collateral reserves, which as we have seen is not possible for WBTC since the reserves are held by a centralized party. Also #862 specifically related to an overflow problem during a potential DAI depeg which is totally different than this report.

These are all the reason why I think this report is invalid.


**sherlock-admin**

 > Escalate for 10 USDC
> 
> Comments from [Bauchibred](https://github.com/Bauchibred) regarding the fact that the present issue #310 is not a duplicate of #817 are correct.
> However, the main argument of this report is that:
> >WBTC may depeg from BTC
> 
> The only fact that WBTC/BTC would depeg is questionable. Since it is issued by a centralized entity (BitGo) it should be treated as trusted, because the only way for it to depeg would be via an error of this centralized entity or if the DAO voted a malicious proposal. See for details: https://www.gemini.com/cryptopedia/wbtc-what-is-wrapped-bitcoin#section-how-w-btc-works
> 
> Additionally, one of the justifications used to support the escalation regarding the deviation threshold mentioned in [#836 ](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/836). There is no guarantee that the TWAP will not stay within the deviation threshold even in the (very) unlikely event that WBTC/BTC depegs if the depegging happens slowly.
> 
> Regarding #862, this is not a duplicate to this report because DAI principal depeg risk comes from depegging of the underlying collateral reserves, which as we have seen is not possible for WBTC since the reserves are held by a centralized party. Also #862 specifically related to an overflow problem during a potential DAI depeg which is totally different than this report.
> 
> These are all the reason why I think this report is invalid.
> 

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**hrishibhat**

@Bauchibred any further comments on the validity of the issue due? 

**Bauchibred**

@hrishibhat I still think issue should be valid and stand on its own, the reply to my escalation was that the idea of “WBTC depegging is not valid”, but I argue that that's wrong, Wrapped BTCs have depegged multiple times in the past, one of the popular instance being after the whole FTX saga, though fair enough that was soBTC, now worth around 7% of what it's supposed to be pegged to.

Note that Chainlink even provides a separate price feed to query the WBTC/BTC price,seen [here](https://data.chain.link/ethereum/mainnet/crypto-other/wbtc-btc).

So I still believe that the price of WBTC/USD for more accuracy should be calculated based on WBTC/BTC and BTC/USD price feeds instead of directly using the BTC/USD feed. 

Additionally [this article](https://thebitcoinmanual.com/articles/why-wrapped-bitcoin-depeg/) from the bitcoin manual, could provide more insight on how and why wrapped bitcoins could depeg. 

**hrishibhat**

Result:
Medium
Has duplicates
This is not a duplicate of #817 
Considering this issue and other duplicates of this issue as a valid medium given the edge case possibility of WBTC de-pegging.

**sherlock-admin**

Escalations have been resolved successfully!

Escalation status:
- [Bauchibred](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/310/#issuecomment-1604772582): accepted
- [twicek](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/310/#issuecomment-1605817251): rejected

# Issue M-5: Inaccurate collateral factor calculation due to missing collateral asset 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/341 

## Found by 
0xeix, Angry\_Mustache\_Man, Bauer, BugHunter101, Dug, PokemonAuditSimulator, TheNaubit, immeas, juancito, mahdikarimi, ravikiran.web3, sakshamguruji, shaka, theOwl
## Summary
The function `collateralFactor()` in the smart contract calculates the collateral factor for the protocol but fails to account for the removal of certain collateral assets. As a result, the total value of the removed collateral assets is not included in the calculation, leading to an inaccurate collateral factor.

## Vulnerability Detail
The `collateralFactor()` function calculates the current collateral factor for the protocol. It iterates through each collateral asset in the system and calculates the total value of all collateral assets in USD.

For each collateral asset, the function retrieves its balance and converts it to a USD value by multiplying it with the asset's price in USD obtained from the corresponding oracle. The balance is adjusted for the decimal precision of the asset. These USD values are accumulated to calculate the totalAssetsUSD.
```solidity
   function collateralFactor() public view override returns (uint256) {
        uint256 totalAssetsUSD = 0;
        for (uint256 i = 0; i < collateral.length; i++) {
            totalAssetsUSD +=
                (((IERC20Upgradeable(collateral[i].token).balanceOf(
                    address(this)
                ) * 1e18) /
                    (10 **
                        IERC20MetadataUpgradeable(collateral[i].token)
                            .decimals())) *
                    collateral[i].oracle.getPriceUSD()) /
                1e18;
        }

        return (totalAssetsUSD * 1e6) / totalSupply();
    }
```
However, when a collateral asset is removed from the collateral list, the `collateralFactor` function fails to account for its absence. This results in an inaccurate calculation of the collateral factor. Specifically, the totalAssetsUSD variable does not include the value of the removed collateral asset, leading to an underestimation of the total collateral value. The function `SellUSSDBuyCollateral()` in the smart contract is used for rebalancing. However, it relies on the collateralFactor calculation, which has been found to be inaccurate. The collateralFactor calculation does not accurately assess the portions of collateral assets to be bought or sold during rebalancing. This discrepancy can lead to incorrect rebalancing decisions and potentially impact the stability and performance of the protocol.
```solidity
    function removeCollateral(uint256 _index) public onlyControl {
        collateral[_index] = collateral[collateral.length - 1];
        collateral.pop();
    }
```
## Impact
As a consequence, the reported collateral factor may be lower than it should be, potentially affecting the risk assessment and stability of the protocol. 

## Code Snippet
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L179-L194

## Tool used

Manual Review

## Recommendation
Ensure accurate calculations and maintain the integrity of the collateral factor metric in the protocol's risk management system.

# Issue M-6: Inconsistency handling of DAI as collateral in the BuyUSSDSellCollateral function 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/515 

## Found by 
0xRobocop, GimelSec, J4de, WATCHPUG, saidam017
## Summary

DAI is the base asset of the `USSD.sol` contract, when a rebalacing needs to occur during a peg-down recovery, collateral is sold for DAI, which then is used to buy USSD in the DAI / USSD uniswap pool. Hence, when DAI is the collateral, this is not sold because there no existe a path to sell DAI for DAI.

## Vulnerability Detail

The above behavior is handled when collateral is about to be sold for DAI, see the comment `no need to swap DAI` ([link to the code](https://github.com/sherlock-audit/2023-05-USSD/blob/6d7a9fdfb1f1ed838632c25b6e1b01748d0bafda/ussd-contracts/contracts/USSDRebalancer.sol#L117-L139)):

```solidity
if (collateralval > amountToBuyLeftUSD) {
   // sell a portion of collateral and exit
   if (collateral[i].pathsell.length > 0) {
       uint256 amountBefore = IERC20Upgradeable(baseAsset).balanceOf(USSD);
       uint256 amountToSellUnits = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * ((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18) / 1e18;
       IUSSD(USSD).UniV3SwapInput(collateral[i].pathsell, amountToSellUnits);
       amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
       DAItosell += (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
   } 
   else {
       // no need to swap DAI
       DAItosell = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * amountToBuyLeftUSD / collateralval;
   }
}

else {
   // @audit-issue Not handling the case where this is DAI as is done above.
   // sell all or skip (if collateral is too little, 5% treshold)
   if (collateralval >= amountToBuyLeftUSD / 20) {
      uint256 amountBefore = IERC20Upgradeable(baseAsset).balanceOf(USSD);
      // sell all collateral and move to next one
      IUSSD(USSD).UniV3SwapInput(collateral[i].pathsell, IERC20Upgradeable(collateral[i].token).balanceOf(USSD));
      amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
      DAItosell += (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
   }
}
```
The problem is in the `else branch` of the first if statement `collateralval > amountToBuyLeftUSD`, which lacks the check `if (collateral[i].pathsell.length > 0)`

## Impact

A re-balancing on a peg-down recovery will fail if the `collateralval` of DAI is less than `amountToBuyLeftUSD` but greater than `amountToBuyLeftUSD / 20` since the DAI collateral does not have a sell path.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/6d7a9fdfb1f1ed838632c25b6e1b01748d0bafda/ussd-contracts/contracts/USSDRebalancer.sol#L130-L139

## Tool used

Manual Review

## Recommendation

Handle the case as is the done on the if branch of `collateralval > amountToBuyLeftUSD`:

```solidity
if (collateral[i].pathsell.length > 0) {
  // Sell collateral for DAI
}
else {
 // No need to swap DAI
}
```




## Discussion

**0xRobocop**

Escalate for 10 USDC

This is not a duplicate of #111 

This issue points to an inconsistency in handling DAI as a collateral during peg-down recovery scenarios. The contract will try to sell DAI, but DAI does not have a sell path, so the transaction will revert.

```solidity
if (collateralval > amountToBuyLeftUSD) {
   // sell a portion of collateral and exit
   if (collateral[i].pathsell.length > 0) {
       uint256 amountBefore = IERC20Upgradeable(baseAsset).balanceOf(USSD);
       uint256 amountToSellUnits = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * ((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18) / 1e18;
       IUSSD(USSD).UniV3SwapInput(collateral[i].pathsell, amountToSellUnits);
       amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
       DAItosell += (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
   } 
   else {
       // no need to swap DAI
       DAItosell = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * amountToBuyLeftUSD / collateralval;
   }
}

else {
   // @audit-issue Not handling the case where this is DAI as is done above.
   // sell all or skip (if collateral is too little, 5% treshold)
   if (collateralval >= amountToBuyLeftUSD / 20) {
      uint256 amountBefore = IERC20Upgradeable(baseAsset).balanceOf(USSD);
      // sell all collateral and move to next one
      IUSSD(USSD).UniV3SwapInput(collateral[i].pathsell, IERC20Upgradeable(collateral[i].token).balanceOf(USSD));
      amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
      DAItosell += (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
   }
}
```

See the inconsistency on the upper `if` and `else` branches. The `else` branch may try to sell DAI, but DAI does not have a sell path.

**sherlock-admin**

 > Escalate for 10 USDC
> 
> This is not a duplicate of #111 
> 
> This issue points to an inconsistency in handling DAI as a collateral during peg-down recovery scenarios. The contract will try to sell DAI, but DAI does not have a sell path, so the transaction will revert.
> 
> ```solidity
> if (collateralval > amountToBuyLeftUSD) {
>    // sell a portion of collateral and exit
>    if (collateral[i].pathsell.length > 0) {
>        uint256 amountBefore = IERC20Upgradeable(baseAsset).balanceOf(USSD);
>        uint256 amountToSellUnits = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * ((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18) / 1e18;
>        IUSSD(USSD).UniV3SwapInput(collateral[i].pathsell, amountToSellUnits);
>        amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
>        DAItosell += (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
>    } 
>    else {
>        // no need to swap DAI
>        DAItosell = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * amountToBuyLeftUSD / collateralval;
>    }
> }
> 
> else {
>    // @audit-issue Not handling the case where this is DAI as is done above.
>    // sell all or skip (if collateral is too little, 5% treshold)
>    if (collateralval >= amountToBuyLeftUSD / 20) {
>       uint256 amountBefore = IERC20Upgradeable(baseAsset).balanceOf(USSD);
>       // sell all collateral and move to next one
>       IUSSD(USSD).UniV3SwapInput(collateral[i].pathsell, IERC20Upgradeable(collateral[i].token).balanceOf(USSD));
>       amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
>       DAItosell += (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore);
>    }
> }
> ```
> 
> See the inconsistency on the upper `if` and `else` branches. The `else` branch may try to sell DAI, but DAI does not have a sell path.

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**hrishibhat**

@ctf-sec 

**hrishibhat**

Result:
Medium
Has duplicates

**sherlock-admin**

Escalations have been resolved successfully!

Escalation status:
- [0xRobocop](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/515/#issuecomment-1605656433): accepted

# Issue M-7: Risk of Incorrect Asset Pricing by StableOracle in Case of Underlying Aggregator Reaching minAnswer 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/598 

## Found by 
Bauchibred, BugBusters, Madalad, RaymondFam, T1MOH, TheNaubit, berlin-101, chalex.eth, kiki\_dev
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

# Issue M-8: `BuyUSSDSellCollateral()` always sells 0 amount if need to sell part of collateral 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/656 

## Found by 
J4de, T1MOH
## Summary
Due to rounding error there is misbehaviour in `BuyUSSDSellCollateral()` function. It results in selling 0 amount of collateral.

## Vulnerability Detail
Suppose the only collateral in protocol is 1 WBTC; 1 WBTC costs 30_000 USD;
UniV3Pool DAI/ USSD has following liquidity: (3000e6 USSD, 2000e18 DAI)
And also USSD is underpriced so call rebalance:
```solidity
    function rebalance() override public {
      uint256 ownval = getOwnValuation(); // it low enough to dive into if statement (see line below) 
      (uint256 USSDamount, uint256 DAIamount) = getSupplyProportion(); // (3000e6 USSD, 2000e18 DAI)
      if (ownval < 1e6 - threshold) {
        // peg-down recovery
        BuyUSSDSellCollateral((USSDamount - DAIamount / 1e12)/2); //  500 * 1e6     = (3000e6 - 2000e18 / 1e12) / 2
```
Take a look into BuyUSSDSellCollateral (follow comments):
```solidity
    function BuyUSSDSellCollateral(uint256 amountToBuy) internal { // 500e6
      CollateralInfo[] memory collateral = IUSSD(USSD).collateralList();
      //uint amountToBuyLeftUSD = amountToBuy * 1e12 * 1e6 / getOwnValuation();
      uint amountToBuyLeftUSD = amountToBuy * 1e12; // 500e18
      uint DAItosell = 0;
      // Sell collateral in order of collateral array
      for (uint256 i = 0; i < collateral.length; i++) {
        // 30_000e18 = 1e8 * 1e18 / 10**8 * 30_000e18 / 1e18
        uint256 collateralval = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * 1e18 / (10**IERC20MetadataUpgradeable(collateral[i].token).decimals()) * collateral[i].oracle.getPriceUSD() / 1e18;
        if (collateralval > amountToBuyLeftUSD) {
          // sell a portion of collateral and exit
          if (collateral[i].pathsell.length > 0) {
            uint256 amountBefore = IERC20Upgradeable(baseAsset).balanceOf(USSD); // 0
            // amountToSellUnits = 1e8 * ((500e18 * 1e18 / 30_000e18) / 1e18) / 1e18 = 1e8 * (0) / 1e18 = 0
            uint256 amountToSellUnits = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * ((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18) / 1e18;
            // and finally executes trade of 0 WBTC
            IUSSD(USSD).UniV3SwapInput(collateral[i].pathsell, amountToSellUnits);
            amountToBuyLeftUSD -= (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore); // 0 = 0 - 0
            DAItosell += (IERC20Upgradeable(baseAsset).balanceOf(USSD) - amountBefore); // 0 += 0
            ...
```
So protocol will not buy DAI and will not sell DAI for USSD in UniswapV3Pool to support peg of USSD to DAI

## Impact
Protocol is not able of partial selling of collateral for token. It block algorithmic pegging of USSD to DAI 

## Code Snippet
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L121

## Tool used

Manual Review, VS Code

## Recommendation
Refactor formula of amountToSellUnits
```solidity
// uint256 amountToSellUnits = (decimals of collateral) * (DAI amount to get for sell) / (price of 1 token of collateral)
uint256 amountToSellUnits = collateral[i].token).decimals() * amountToBuyLeftUSD / collateral[i].oracle.getPriceUSD()
```




## Discussion

**T1MOH593**

Escalate for 10 USDC

This is not a duplicate of #111
This report describes that partially collateral can't be sold, because `amountToSellUnits` is 0 due to rounding issue. Noticed #183 is similar to my issue

**sherlock-admin**

 > Escalate for 10 USDC
> 
> This is not a duplicate of #111
> This report describes that partially collateral can't be sold, because `amountToSellUnits` is 0 due to rounding issue. Noticed #183 is similar to my issue

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**ctf-sec**

I agree this issue and #183 are not duplicate of #111 and can be grouped together as a new valid medium, will see if this is a duplicate of other issue

**hrishibhat**

Result:
Medium
Has duplicates


**sherlock-admin**

Escalations have been resolved successfully!

Escalation status:
- [T1MOH593](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/656/#issuecomment-1604582674): accepted

# Issue M-9: Using the collateral assets' oracle price at 100% of its value to mint USSD without a fee can be used for arbitrage. 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/836 

## Found by 
WATCHPUG
## Summary

Allowing the users to mint USSD using the collateral assets, at 100% of its value based on the oracle price without a fee can easily be exploited by the arbitragers.

## Vulnerability Detail

The Oracle price can not be trusted as the real-time price.

For example, the BTC/USD and ETH/USD price feeds on miannet have a "Deviation threshold" of 0.5%, meaning that the price will only be updated once the price movement exceeds 0.5% within the heartbeat period.

Say if the previous price point for WETH is 1000 USD, the price will only be updated once the price goes up to more than 1005 USD or down to less than 995 USD.

## Impact

When the market price of WETH is lower than the oracle price, it is possible to mint 1000 USSD by using 1 WETH and selling it to DAI, causing the quality of the collateral for USSD to continuously decrease and the value to be leaked to the arbitragers.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L150-L173

## Tool used

Manual Review

## Recommendation

Consider adding a minting fee of 0.5% to 1% (should be higher than the deviation).



## Discussion

**0xRobocop**

Escalate for 10 USDC

This is not an issue, it assumes that a "real-time" price exists which is theoretically impossible. In reality there is no way to value a collateral precisely to a "real-time" price because this "price" does not exists and the markets are aligned thanks to arbitrageurs. 

We cannot say that the chainlink price (if chainlink is behaving properly and contract consumes the prices safely) is below or above the "market-price", because there is no such "market-price", what we can say is that some market has a different price than chainlink's oracle. For example the ETH / DAI uniswap pool may have the price of 1 ETH for 996 DAI and chainlink's price may be 1 ETH for 1000 DAI. Watson argues that this scenario will:

`cause the quality of the collateral for USSD to continuously decrease and the value to be leaked to the arbitragers.`

Which is not true, what will happen is the next:

- 1. User will send 1 ETH to the USSD protocol and receive 1000 USSD.
- 2. User will change 1000 USSD for 1000 DAI.
- 3. User will buy ETH in uniswap with the 1000 DAI and receive 1.004 ETH, driving up the price of ETH in uniswap.
- 4. User will repeat the process until the uniswap price is equal to the chainlink price and the arbitrage is no longer possible.
- 5. ETH price increased in the "below-price" market, making the ETH collateral of the USSD protocol more valuable across different markets.



**sherlock-admin**

 > Escalate for 10 USDC
> 
> This is not an issue, it assumes that a "real-time" price exists which is theoretically impossible. In reality there is no way to value a collateral precisely to a "real-time" price because this "price" does not exists and the markets are aligned thanks to arbitrageurs. 
> 
> We cannot say that the chainlink price (if chainlink is behaving properly and contract consumes the prices safely) is below or above the "market-price", because there is no such "market-price", what we can say is that some market has a different price than chainlink's oracle. For example the ETH / DAI uniswap pool may have the price of 1 ETH for 996 DAI and chainlink's price may be 1 ETH for 1000 DAI. Watson argues that this scenario will:
> 
> `cause the quality of the collateral for USSD to continuously decrease and the value to be leaked to the arbitragers.`
> 
> Which is not true, what will happen is the next:
> 
> - 1. User will send 1 ETH to the USSD protocol and receive 1000 USSD.
> - 2. User will change 1000 USSD for 1000 DAI.
> - 3. User will buy ETH in uniswap with the 1000 DAI and receive 1.004 ETH, driving up the price of ETH in uniswap.
> - 4. User will repeat the process until the uniswap price is equal to the chainlink price and the arbitrage is no longer possible.
> - 5. ETH price increased in the "below-price" market, making the ETH collateral of the USSD protocol more valuable across different markets.
> 
> 

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**twicek**

Escalate for 10 USDC

I agree with the comments made by 0xRobocop, this report describe a common scenario that lead to an arbitrage opportunity, which is not an issue.
Hitting the deviation threshold will lead for the price to be updated earlier than usual which will naturally lead to the arbitrage opportunity described by 0xRobocop. Adding a minting fee could actually be more detrimental since it would prevent arbitrager from getting the USSD / DAI Pool to equilibrium.

**sherlock-admin**

 > Escalate for 10 USDC
> 
> I agree with the comments made by 0xRobocop, this report describe a common scenario that lead to an arbitrage opportunity, which is not an issue.
> Hitting the deviation threshold will lead for the price to be updated earlier than usual which will naturally lead to the arbitrage opportunity described by 0xRobocop. Adding a minting fee could actually be more detrimental since it would prevent arbitrager from getting the USSD / DAI Pool to equilibrium.

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**ctf-sec**

Agree with the escalation.

**hrishibhat**

Result:
Medium
Unique
Considering this a valid medium. 
Lead Watson comment:
>  comment is basically describing the arbitrage CAN happen, the missing part there is ETH/DAI is a much deeper pool than USSD/DAI, the USSD/DAI pool will suffer a much bigger damage before the arbitrage opportunity disappears. This is not a common/natural scenario


**sherlock-admin**

Escalations have been resolved successfully!

Escalation status:
- [0xRobocop](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/836/#issuecomment-1605228222): rejected
- [twicek](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/836/#issuecomment-1605860157): rejected

# Issue M-10: If collateral factor is high enough, flutter ends up being out of bounds 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/889 

## Found by 
J4de, neumo
## Summary
In `USSDRebalancer` contract, function `SellUSSDBuyCollateral` will revert everytime a rebalance calls it, provided the collateral factor is greater than all the elements of the `flutterRatios` array.

## Vulnerability Detail
Function `SellUSSDBuyCollateral` calculates `flutter` as the lowest index of the `flutterRatios` array for which the collateral factor is smaller than the flutter ratio.
```solidity
uint256 cf = IUSSD(USSD).collateralFactor();
uint256 flutter = 0;
for (flutter = 0; flutter < flutterRatios.length; flutter++) {
	if (cf < flutterRatios[flutter]) {
	  break;
	}
}
```
The problem arises when, if collateral factor is greater than all flutter values, after the loop `flutter = flutterRatios.length`.

This `flutter` value is used afterwards here:
```solidity
...
if (collateralval * 1e18 / ownval < collateral[i].ratios[flutter]) {
  portions++;
}
...
```
 And here:
 ```solidity
...
if (collateralval * 1e18 / ownval < collateral[i].ratios[flutter]) {
  if (collateral[i].token != uniPool.token0() || collateral[i].token != uniPool.token1()) {
	// don't touch DAI if it's needed to be bought (it's already bought)
	IUSSD(USSD).UniV3SwapInput(collateral[i].pathbuy, daibought/portions);
  }
}
...
```

As we can see in the tests of the project, the flutterRatios array and the collateral ratios array are set to be of the same length, so if flutter = flutterRatios.length, any call to that index in the `ratios` array will revert with an index out of bounds.

## Impact
High, when the collateral factor reaches certain level, a rebalance that calls `SellUSSDBuyCollateral` will always revert.

## Code Snippet
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L178-L184

## Tool used
Manual review.


## Recommendation
When checking `collateral[i].ratios[flutter]` always check first that flutter is `< flutterRatios.length`.





## Discussion

**neumoxx**

Escalate for 10 USDC
The issue is marked as duplicate of #940, and in that issue there's a comment from the judge that states `This is an admin input, requires admin error to cause problems`. The issue does not depend on an admin input to arise. The flutter ratios are set in the tests according to values mentioned in the whitepaper: https://github.com/sherlock-audit/2023-05-USSD/blob/6d7a9fdfb1f1ed838632c25b6e1b01748d0bafda/ussd-contracts/test/USSDsimulator.test.js#L391-L393. 
The collateral factor:
https://github.com/sherlock-audit/2023-05-USSD/blob/6d7a9fdfb1f1ed838632c25b6e1b01748d0bafda/ussd-contracts/contracts/USSD.sol#L179-L191
can grow beyond the last value of the flutter ratios array and that would make the `SellUSSDBuyCollateral` function to revert.

**sherlock-admin**

 > Escalate for 10 USDC
> The issue is marked as duplicate of #940, and in that issue there's a comment from the judge that states `This is an admin input, requires admin error to cause problems`. The issue does not depend on an admin input to arise. The flutter ratios are set in the tests according to values mentioned in the whitepaper: https://github.com/sherlock-audit/2023-05-USSD/blob/6d7a9fdfb1f1ed838632c25b6e1b01748d0bafda/ussd-contracts/test/USSDsimulator.test.js#L391-L393. 
> The collateral factor:
> https://github.com/sherlock-audit/2023-05-USSD/blob/6d7a9fdfb1f1ed838632c25b6e1b01748d0bafda/ussd-contracts/contracts/USSD.sol#L179-L191
> can grow beyond the last value of the flutter ratios array and that would make the `SellUSSDBuyCollateral` function to revert.

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**ctf-sec**

[here](https://github.com/sherlock-audit/2023-05-USSD/blob/6d7a9fdfb1f1ed838632c25b6e1b01748d0bafda/ussd-contracts/contracts/USSDRebalancer.sol#L62)

```solidity
    function setFlutterRatios(uint256[] calldata _flutterRatios) public onlyControl {
      flutterRatios = _flutterRatios;
    }
```

flutterRatios can be adjusted by admin

Valid low

**hrishibhat**

Result:
Medium
Has duplicates
This is a valid issue where the rebalance reverts in certain conditions due to the unexpected final loop flutter values. 

**sherlock-admin**

Escalations have been resolved successfully!

Escalation status:
- [neumoxx](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/889/#issuecomment-1605375095): accepted

# Issue M-11: Lack of Redeem Feature 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/958 

## Found by 
0xMosh, 0xRobocop, BugBusters, WATCHPUG, ast3ros, ctf\_sec, juancito, sashik\_eth, shealtielanz, shogoki, the\_endless\_sea, ustas
## Summary

## Vulnerability Detail

The whitepaper mentions a redeeming feature that allows the user to redeem USSD for DAI (see section 4 "Mint and redeem"), but it is currently missing from the implementation.

Although there is a "redeem" boolean in the collateral settings, there is no corresponding feature that enables the redemption of USSD to any of the underlying collateral assets.

## Impact

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/interfaces/IUSSDRebalancer.sol#L13-L21

## Tool used

Manual Review

## Recommendation

Revise the whitepaper and docs to reflect the fact that there is no redeem function or add a redeem function.



## Discussion

**securitygrid**

Escalate for 10 USDC
This is valid low/info.
So far, lacking redeem is no impact.

**sherlock-admin**

 > Escalate for 10 USDC
> This is valid low/info.
> So far, lacking redeem is no impact.

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**0xJuancito**

Escalate for 10 USDC

This is a valid High. 

--- 

Escalating the comment:

> This is valid low/info.
> So far, lacking redeem is no impact.

--- 

The impact is explained on the whitepaper and quoted on the "Vulnerability Detail" section from issue https://github.com/sherlock-audit/2023-05-USSD-judging/issues/218:

As per the [USSD whitepaper](https://github.com/USSDofficial/ussd-whitepaper/blob/main/whitepaper.pdf):

> If there is positive DAI balance in the collateral, USSD contract can provide
DAI for equal amount of USSD in return (that would be burned, contracting
supply).

The importance of this is said here:

> Ability to mint and redeem USSD for DAI could serve as incentives to rebalance the coin when this is economically viable

And the most important feature is to have a mechanism to "help USSD recover in negative scenarios":

> These methods also could be used to help USSD recover in negative scenarios:
if USSD value falls below 1 DAI and there are less than 1 DAI reserves per USSD
to refill the reserves allowing the USSD to recover it’s price by reducing supply

**sherlock-admin**

 > Escalate for 10 USDC
> 
> This is a valid High. 
> 
> --- 
> 
> Escalating the comment:
> 
> > This is valid low/info.
> > So far, lacking redeem is no impact.
> 
> --- 
> 
> The impact is explained on the whitepaper and quoted on the "Vulnerability Detail" section from issue https://github.com/sherlock-audit/2023-05-USSD-judging/issues/218:
> 
> As per the [USSD whitepaper](https://github.com/USSDofficial/ussd-whitepaper/blob/main/whitepaper.pdf):
> 
> > If there is positive DAI balance in the collateral, USSD contract can provide
> DAI for equal amount of USSD in return (that would be burned, contracting
> supply).
> 
> The importance of this is said here:
> 
> > Ability to mint and redeem USSD for DAI could serve as incentives to rebalance the coin when this is economically viable
> 
> And the most important feature is to have a mechanism to "help USSD recover in negative scenarios":
> 
> > These methods also could be used to help USSD recover in negative scenarios:
> if USSD value falls below 1 DAI and there are less than 1 DAI reserves per USSD
> to refill the reserves allowing the USSD to recover it’s price by reducing supply

You've created a valid escalation for 10 USDC!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**abhishekvispute**

well this is what protocol team said that time regarding redeem 
that it is intentional 

![image](https://github.com/sherlock-audit/2023-05-USSD-judging/assets/46760063/a3c4378e-eab2-4a1a-946a-f5107f756dbc)


**ctf-sec**

> well this is what protocol team said that time regarding redeem that it is intentional
> 
> ![image](https://user-images.githubusercontent.com/46760063/248562171-a3c4378e-eab2-4a1a-946a-f5107f756dbc.png)

valid low based on the sponsor's feedback

**hrishibhat**

Result:
Medium
Has duplicates
This issue can be considered a valid medium based on the Whitepaper description of the importance of having a the redeem function.

**sherlock-admin**

Escalations have been resolved successfully!

Escalation status:
- [0xJuancito](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/958/#issuecomment-1604979329): accepted
- [securitygrid](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/958/#issuecomment-1604576035): rejected

