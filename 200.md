T1MOH

high

# Zero address is used as StableOracleWETH in StableOracleDAI

## Summary
StableOracleDAI will not work, due to revert in `getPriceUSD()` function

## Vulnerability Detail
Zero address is passed in constructor
```solidity
contract StableOracleDAI is IStableOracle {
    AggregatorV3Interface priceFeedDAIETH;
    IStaticOracle DAIEthOracle;
    IStableOracle ethOracle;

    constructor() {
        priceFeedDAIETH = AggregatorV3Interface(
            0x773616E4d11A78F511299002da57A0a94577F1f4
        );
        DAIEthOracle = IStaticOracle(
            0x982152A6C7f732Ec7C9EA998dDD9Ebde00Dfa16e
        );
        ethOracle = IStableOracle(0x0000000000000000000000000000000000000000); // TODO: WETH oracle price
    }
```

And then ethOracle is called:
```solidity
    function getPriceUSD() external view override returns (uint256) {
        ...

        uint256 wethPriceUSD = ethOracle.getPriceUSD();

        ...
        return
            (wethPriceUSD * 1e18) /
            ((DAIWethPrice + uint256(price) * 1e10) / 2);
    }
```

## Impact
It will block using DAI as collateral in USSD.sol

## Code Snippet
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L30

## Tool used

Manual Review

## Recommendation
Refactor constructor:
```solidity
    constructor(address _ethOracle) {
        priceFeedDAIETH = AggregatorV3Interface(
            0x773616E4d11A78F511299002da57A0a94577F1f4
        );
        DAIEthOracle = IStaticOracle(
            0x982152A6C7f732Ec7C9EA998dDD9Ebde00Dfa16e
        );
        ethOracle = IStableOracle(_ethOracle);
    }
```
