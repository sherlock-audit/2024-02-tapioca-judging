Winning Cobalt Barracuda

medium

# `getCollateral` and `getAsset` functions of the AssetTotsDaiLeverageExecutor contract decode data incorrectly

## Summary
See vulnerability detail
## Vulnerability Detail
In AssetTotsDaiLeverageExecutor contract, `getCollateral` function decodes the data before passing it to `_swapAndTransferToSender` function.
```solidity=
SLeverageSwapData memory swapData = abi.decode(data, (SLeverageSwapData));
uint256 daiAmount =
    _swapAndTransferToSender(false, assetAddress, daiAddress, assetAmountIn, swapData.swapperData);
```
However, `_swapAndTransferToSender` will decode this data again to obtain the swapperData:
```solidity=
function _swapAndTransferToSender(
    bool sendBack,
    address tokenIn,
    address tokenOut,
    uint256 amountIn,
    bytes memory data
) internal returns (uint256 amountOut) {
    SLeverageSwapData memory swapData = abi.decode(data, (SLeverageSwapData));
    ...
```
The redundant decoding will cause the data to not align as expected, which is different from `SimpleLeverageExecutor.getCollateral()` function ([code snippet](https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/leverage/SimpleLeverageExecutor.sol#L46))
## Impact
`getCollateral` and `getAsset` of AssetTotsDaiLeverageExecutor will not work as intended due to incorrectly decoding data.
## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/leverage/AssetTotsDaiLeverageExecutor.sol#L53-L55
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/leverage/AssetTotsDaiLeverageExecutor.sol#L88-L91

## Tool used

Manual Review

## Recommendation
The AssetTotsDaiLeverageExecutor contract should pass data directly to `_swapAndTransferToSender`, similar to the SimpleLeverageExecutor contract