Modern Mandarin Wasp

medium

# BBLeverage::sellCollateral is unusable due to wrong asset deposit attempt in YieldBox

## Summary
`sellCollateral` enable to leverage up on a borrow position in a BigBang market. However the endpoint is unusable as is due to `collateralId` used to deposit in YieldBox, instead of `assetId`

## Vulnerability Detail
We can see here that after withdrawing `collateral`, and swapping it for `asset`, `BBLeverage::sellCollateral` attempts to deposit `collateralId` into YieldBox:
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L149

Which will always revert, since at that point we always have `asset` and not `collateral`.

This function is thus unusable

## Impact
The function `BBLeverage::sellCollateral` will always revert and is unusable

## Code Snippet

## Tool used

Manual Review

## Recommendation
Change the deposit to use `assetId`, as is correctly done in SGLLeverage:
```diff
-        yieldBox.depositAsset(collateralId, address(this), address(this), 0, memoryData.shareOut); // TODO Check for rounding attack?
+        yieldBox.depositAsset(assetId, address(this), address(this), 0, memoryData.shareOut); // TODO Check for rounding attack?
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLeverage.sol#L135