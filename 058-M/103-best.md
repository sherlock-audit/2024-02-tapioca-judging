Vast Bronze Salamander

medium

# sellCollateral() using incorrect parameters when calling getAsset

## Summary
sellCollateral() using incorrect parameters when calling getAsset

## Vulnerability Detail
`BBLeverage.sellCollateral()` the code is follows:

```solidity
    function sellCollateral(address from, uint256 share, bytes calldata data)
        external
        optionNotPaused(PauseType.LeverageSell)
        solvent(from, false)
        notSelf(from)
        returns (uint256 amountOut)
    {
...
        amountOut = leverageExecutor.getAsset(
@>          assetId, address(collateral), address(asset), memoryData.leverageAmount, from, data
        );
        memoryData.shareOut = yieldBox.toShare(assetId, amountOut, false);
        address(asset).safeApprove(address(yieldBox), type(uint256).max);
```

In the function call `getAsset(assetId, address(collateral)...)`, the second parameter is passed as `collateral`, but it should actually be `asset`.

```solidity
interface ILeverageExecutor {
..
    function getAsset(
        uint256 assetId,
@>      address assetAddress,
        address collateralAddress,
        uint256 collateralAmountIn,
        address from,
        bytes calldata data
    ) external returns (uint256 assetAmountOut); //used for sellCollateral
```

## Impact
Incorrect usage of `getAsset()` can lead to exchange failures.

## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L144

## Tool used

Manual Review

## Recommendation
```diff
    function sellCollateral(address from, uint256 share, bytes calldata data)
        external
        optionNotPaused(PauseType.LeverageSell)
        solvent(from, false)
        notSelf(from)
        returns (uint256 amountOut)
    {
        if (address(leverageExecutor) == address(0)) {
            revert LeverageExecutorNotValid();
        }
        _allowedBorrow(from, share);
        _removeCollateral(from, address(this), share);

        _SellCollateralMemoryData memory memoryData;

        (, memoryData.obtainedShare) =
            yieldBox.withdraw(collateralId, address(this), address(leverageExecutor), 0, share);
        memoryData.leverageAmount = yieldBox.toAmount(collateralId, memoryData.obtainedShare, false);
        amountOut = leverageExecutor.getAsset(
-           assetId, address(collateral), address(asset), memoryData.leverageAmount, from, data
+           assetId, address(asset), address(collateral), memoryData.leverageAmount, from, data
        );
```