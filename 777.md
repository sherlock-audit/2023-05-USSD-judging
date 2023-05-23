WATCHPUG

high

# Lack of access control for `mintRebalancer()` and `burnRebalancer()`

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