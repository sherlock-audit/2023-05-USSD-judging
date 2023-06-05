Kose

high

# Because of missing slippage parameter, mintForToken() can be front-runned

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
