Winning Cobalt Barracuda

medium

# `leverageAmount` is incorrect in  `SGLLeverage.sellCollateral` function due to calculation based on the new states of YieldBox after withdrawal

## Summary
See vulnerability detail
## Vulnerability Detail
`SGLLeverage.sellCollateral` function attempts to remove the user's collateral in shares of YieldBox, then withdraws those collateral shares to collect collateral tokens. Subsequently, the received collateral tokens can be used to swap for asset tokens.

However, the `leverageAmount` variable in this function does not represent the actual withdrawn tokens from the provided shares because it is calculated after the withdrawal.
```solidity=
yieldBox.withdraw(collateralId, address(this), address(leverageExecutor), 0, calldata_.share);
uint256 leverageAmount = yieldBox.toAmount(collateralId, calldata_.share, false);

amountOut = leverageExecutor.getAsset(
    assetId, address(collateral), address(asset), leverageAmount, calldata_.from, calldata_.data
);
```
`yieldBox.toAmount` after withdrawal may return different from the actual withdrawn token amount, because the states of YieldBox has changed. Because the token amount is calculated with rounding down in YieldBox, `leverageAmount` will be higher than the actual withdrawn amount.

For example, before the withdrawal, YieldBox had 100 total shares and 109 total tokens. Now this function attempt to withdraw 10 shares (`calldata_.share` = 10)
**-> the actual withdrawn amount = 10 * 109 / 100 = 10 tokens**
After that, leverageAmount will be calculated based on the new yieldBox's total shares and total tokens
**-> leverageAmount = 10 * (109 - 10) / (100 - 10) = 11 tokens**

The same vulnerability exists in `BBLeverage.sellCollateral` function.

## Impact
Because `leverageAmount` can be higher than the actual withdrawn collateral tokens, `leverageExecutor.getAsset()` will revert due to not having enough tokens in the contract to pull. This results in a DOS of `sellCollateral`, break this functionality.
## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLeverage.sol#L127-L128
## Tool used

Manual Review

## Recommendation
`leverageAmount` should be obtained from the return value of `YieldBox.withdraw`:
```solidity=
(uint256 leverageAmount, ) = yieldBox.withdraw(collateralId, address(this), address(leverageExecutor), 0, calldata_.share);
```