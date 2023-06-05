GimelSec

medium

# The rounding error in `USSD.calculateMint` could cause the loss of funds.

## Summary

`USSD.mintForToken` uses `USSD.calculateMint` to calculate the amount of received USSD. However, there is a rounding error in the calculation. It may cause the user to receive a smaller amount of USSD.

## Vulnerability Detail

The calculation in `USSD.calculateMint` could suffer from the rounding error.
https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L172
```solidity
    function calculateMint(address _token, uint256 _amount) public view returns (uint256 stableCoinAmount) {
        uint256 assetPrice = collateral[getCollateralIndex(_token)].oracle.getPriceUSD();
        return (((assetPrice * _amount) / 1e18) * (10 ** decimals())) / (10 ** IERC20MetadataUpgradeable(_token).decimals());
    }
```

If `(assetPrice * _amount)` is less than 1e18, the result has rounding error.
```solidity
// setting
assetPrice = 0.3 * 1e18
_amount = 3
decimals() = 6
IERC20MetadataUpgradeable(_token).decimals() = 5

// and the result is
(((assetPrice * _amount) / 1e18) * (10 ** decimals())) / (10 ** IERC20MetadataUpgradeable(_token).decimals()) =
((0.3 * 1e18 * 3) / 1e18) * 1e6 / 1e5 = 0

// but if you change the order of the calculation. It shouldn’t be zero
(((assetPrice * _amount)) * (10 ** decimals())) / (10 ** IERC20MetadataUpgradeable(_token).decimals())  / 1e18 = 
(0.3 * 1e18 * 3) * 1e6 / 1e5 / 1e18 = 9
```

## Impact

The round error affects the amount of the received USSD. Users could suffer from the loss of the funds.

## Code Snippet

https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L172


## Tool used

Manual Review

## Recommendation

Modify the calculation to prevent the rounding error.
```diff
    function calculateMint(address _token, uint256 _amount) public view returns (uint256 stableCoinAmount) {
        uint256 assetPrice = collateral[getCollateralIndex(_token)].oracle.getPriceUSD();
-       return (((assetPrice * _amount) / 1e18) * (10 ** decimals())) / (10 ** IERC20MetadataUpgradeable(_token).decimals());
+       return ((assetPrice * _amount) * (10 ** decimals())) / (10 ** IERC20MetadataUpgradeable(_token).decimals())  / 1e18;
    }
```
