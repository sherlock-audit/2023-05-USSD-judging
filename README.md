# Issue H-1: `USSDRebalancer.sol#SellUSSDBuyCollateral` the check of whether collateral is DAI is wrong 

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

# Issue H-2: The getOwnValuation() function contains errors in the price calculation 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/222 

## Found by 
0xpinky, AlexCzm, Bauer, J4de, T1MOH, carrotsmuggler, kiki\_dev, peanuts, sam\_gmk, sashik\_eth, simon135, theOwl, twicek, warRoom
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

# Issue H-3: The price from `StableOracleDAI` is returned with the incorrect number of decimals 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/236 

## Found by 
0xPkhatri, 0xRobocop, 0xStalin, 0xeix, 0xlmanini, Bahurum, Brenzee, Dug, G-Security, J4de, PNS, Proxy, SanketKogekar, T1MOH, Vagner, WATCHPUG, ast3ros, ctf\_sec, immeas, juancito, kutugu, n33k, nobody2018, peanuts, pengun, qbs, qpzm, saidam017, sam\_gmk, sashik\_eth, twicek
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

# Issue H-4: Price calculation susceptible to flashloan exploits 

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

# Issue H-5: Wrong computation of the amountToSellUnit variable 

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

# Issue H-6: Not using slippage parameter or deadline while swapping on UniswapV3 

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

# Issue H-7: Lack of access control for `mintRebalancer()` and `burnRebalancer()` 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/777 

## Found by 
0x2e, 0xAzez, 0xHati, 0xMojito, 0xPkhatri, 0xRobocop, 0xSmartContract, 0xStalin, 0xeix, 0xyPhilic, 14si2o\_Flint, AlexCzm, Angry\_Mustache\_Man, Aymen0909, Bahurum, Bauchibred, Bauer, BlockChomper, Brenzee, BugBusters, BugHunter101, Delvir0, DevABDee, Dug, Fanz, GimelSec, HonorLt, J4de, JohnnyTime, Juntao, Kodyvim, Kose, Lilyjjo, Madalad, Nyx, PokemonAuditSimulator, RaymondFam, Saeedalipoor01988, SanketKogekar, Schpiel, SensoYard, T1MOH, TheNaubit, Tricko, VAD37, Vagner, WATCHPUG, \_\_141345\_\_, anthony, ast3ros, auditsea, berlin-101, blackhole, blockdev, carrotsmuggler, chainNue, chalex.eth, cjm00n, coincoin, coryli, ctf\_sec, curiousapple, dacian, evilakela, georgits, giovannidisiena, immeas, innertia, jah, juancito, kie, kiki\_dev, kutugu, lil.eth, m4ttm, mahdikarimi, mrpathfindr, n33k, neumo, ni8mare, nobody2018, pavankv241, pengun, qbs, qckhp, qpzm, ravikiran.web3, saidam017, sam\_gmk, sashik\_eth, shaka, shealtielanz, shogoki, simon135, slightscan, smiling\_heretic, tallo, theOwl, the\_endless\_sea, toshii, tsvetanovv, tvdung94, twcctop, twicek, vagrant, ver0759, warRoom, whiteh4t9527, ww4tson, yy
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

# Issue H-8: Uniswap v3 pool token balance proportion does not necessarily correspond to the price, and it is easy to manipulate. 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/808 

## Found by 
WATCHPUG
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

# Issue H-9: Wrong Oracle feed addresses 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/817 

## Found by 
0xGusMcCrae, 0xHati, 0xPkhatri, 0xRobocop, 0xStalin, 0xeix, 0xlmanini, 0xyPhilic, 14si2o\_Flint, ADM, Aymen0909, Bahurum, Bauchibred, Bauer, BenRai, Brenzee, BugHunter101, Delvir0, DevABDee, Dug, G-Security, GimelSec, HonorLt, J4de, JohnnyTime, Juntao, Kirkeelee, Kodyvim, Kose, Lilyjjo, Madalad, PNS, PTolev, PokemonAuditSimulator, Proxy, RaymondFam, Saeedalipoor01988, SaharDevep, Schpiel, SensoYard, T1MOH, TheNaubit, Vagner, Viktor\_Cortess, WATCHPUG, \_\_141345\_\_, ashirleyshe, ast3ros, berlin-101, blockdev, chainNue, chalex.eth, ck, ctf\_sec, curiousapple, dacian, evilakela, giovannidisiena, immeas, innertia, juancito, kie, kiki\_dev, kutugu, lil.eth, martin, mrpathfindr, neumo, ni8mare, nobody2018, peanuts, pengun, qpzm, ravikiran.web3, saidam017, sakshamguruji, sam\_gmk, sashik\_eth, shaka, shogoki, simon135, theOwl, the\_endless\_sea, toshii, twicek, ustas, whiteh4t9527
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

# Issue H-10: Using WBGL as collateral poses systemic risk to the USSD stablecoin. 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/822 

## Found by 
0xSmartContract, CodeFoxInc, WATCHPUG, \_\_141345\_\_, ctf\_sec, giovannidisiena, juancito, n33k, qckhp, sashik\_eth, shaka, simon135
## Summary

WBGL has low liquidity and accepting it as collateral without limit can lead to severe depeg of the stablecoin, allowing attackers to manipulate the price and dump other collateral assets for profit. This is due to the ease of manipulating the price of WBGL.

## Vulnerability Detail

WBGL is a small-cap token with very thin liquidity.

Accepting WBGL as collateral without a cap, ie, allows users to mint as much as they want at the easily manipulated price, which can result in severe depeg of the stablecoin. 

## Impact

Specifically, an attacker could:

1. manipulate the price of WBGL to a higher price;
2. mint and dump part of the USSD for DAI;
3. trigger a rebalance, sell the collateral assets to pump the USSD price;
4. dump the rest from step 2 for more DAI.

The root cause of this issue is the thin liquidity of WBGL, which makes it easy to manipulate its price.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBGL.sol#L24-L39

## Tool used

Manual Review

## Recommendation

Remove WBGL from the list of collateral. Or, adding a cap for using WBGL as the collateral to mint USSD, say, no more than 1/10th of the liquidity of WBGL.

# Issue H-11: Oracle price should be denominated in DAI instead of USD 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/909 

## Found by 
Bahurum, Bauer, Brenzee, Viktor\_Cortess, WATCHPUG, ast3ros, juancito, pengun
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

# Issue H-12: Lack of Redeem Feature 

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
Aymen0909, GimelSec, Kose, cryptostellar5, qbs, shealtielanz
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

# Issue M-3: rebalance process incase of  selling the collateral, could revert because of underflow calculation 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/111 

## Found by 
0xHati, 0xRobocop, Dug, GimelSec, J4de, Juntao, PokemonAuditSimulator, T1MOH, WATCHPUG, XDZIBEC, ast3ros, nobody2018, saidam017, toshii, tsvetanovv, twicek
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

# Issue M-4: `USSD.sol#initialize` mint without deposit collateral 

Source: https://github.com/sherlock-audit/2023-05-USSD-judging/issues/177 

## Found by 
0xGusMcCrae, BugBusters, J4de, immeas, sakshamguruji, sam\_gmk
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

# Issue M-6: Using the collateral assets' oracle price at 100% of its value to mint USSD without a fee can be used for arbitrage. 

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

