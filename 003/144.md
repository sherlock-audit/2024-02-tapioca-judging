Zealous Pineapple Duck

high

# Operation residual is lost for the user of BBLeverage's and SGLLeverage's `sellCollateral()`

## Summary

`sellCollateral()` sells the specified amount of collateral, then repays the debt. The proceedings can cover the debt with some excess amount. Moreover, as the market swaps are involved and the actual proceedings amount isn't known in advance, while there is an over collaterization present in any healthy loan, the usual going concern situation is to sell slightly more when the goal is to pay off the debt.

Currently all the remainder of collateral sale will be left with the contract. It can be immediately stolen via `skim` option.

## Vulnerability Detail

Core issue is that all sale proceeding are now stay with the contract, but remainder handling logic isn't present, so the leftover amounts, which can be arbitrary big, are lost for user and left on the contract balance.

These funds are unaccounted and can be skimmed by a back-running attacker.

## Impact

There is the only prerequisite of having a material amount of extra funds from collateral sale, which can happen frequently, so the probability is medium/high. Affected funds can be arbitrary big and are lost for the user, can be stolen by any attacker immediately via back-running. 

Likelihood: Medium/High + Impact: High = Severity: High.

## Code Snippet

All collateral sale proceedings are left with the contract instead of going to `from`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L126-L150

```solidity
    function sellCollateral(address from, uint256 share, bytes calldata data)
        ...
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
            assetId, address(collateral), address(asset), memoryData.leverageAmount, from, data
        );
        memoryData.shareOut = yieldBox.toShare(assetId, amountOut, false);
        address(asset).safeApprove(address(yieldBox), type(uint256).max);
>>      yieldBox.depositAsset(collateralId, address(this), address(this), 0, memoryData.shareOut); // TODO Check for rounding attack?
        address(asset).safeApprove(address(yieldBox), 0);
```

LeverageExecutors return the whole proceedings to sender, i.e. SGL or BB contract:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/leverage/AssetTotsDaiLeverageExecutor.sol#L91-L92

```solidity
        assetAmountOut = _swapAndTransferToSender(true, daiAddress, assetAddress, obtainedDai, swapData.swapperData);
    }
```

Previously it remained with `from` (the code is from the old snapshot):

https://github.com/Tapioca-DAO/Tapioca-bar/blob/f15aa5143f3435b6efbcc19419d1a3b1d1388bdb/contracts/markets/leverage/AssetToRethLeverageExecutor.sol#L108-L114

```
        yieldBox.depositAsset(
            collateralId,
            address(this),
>>          from,
            collateralAmountOut,
            0
        );
```

So in the current implementation assets are left with the contract and `memoryData.shareOut - memoryData.shareOwed` residual will be removed from and not reimbursed to the `from`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L144-L161

```solidity
>>      amountOut = leverageExecutor.getAsset(
            assetId, address(collateral), address(asset), memoryData.leverageAmount, from, data
        );
        memoryData.shareOut = yieldBox.toShare(assetId, amountOut, false);
        address(asset).safeApprove(address(yieldBox), type(uint256).max);
>>      yieldBox.depositAsset(collateralId, address(this), address(this), 0, memoryData.shareOut); // TODO Check for rounding attack?
        address(asset).safeApprove(address(yieldBox), 0);

        memoryData.partOwed = userBorrowPart[from];
        memoryData.amountOwed = totalBorrow.toElastic(memoryData.partOwed, true);
        memoryData.shareOwed = yieldBox.toShare(assetId, memoryData.amountOwed, true);
>>      if (memoryData.shareOwed <= memoryData.shareOut) {
            _repay(from, from, memoryData.partOwed);
        } else {
            //repay as much as we can
            uint256 partOut = totalBorrow.toBase(amountOut, false);
            _repay(from, from, partOut);
        }
```

These funds will be frozen with the contract as the system accounting isn't updated either to reflect injection of `memoryData.shareOut - memoryData.shareOwed` amount of `asset`.

This way anyone can steal them via `skim` option:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBCommon.sol#L128-L138

```solidity
    function _addTokens(address from, uint256 _tokenId, uint256 share, uint256 total, bool skim) internal {
        if (skim) {
>>          require(share <= yieldBox.balanceOf(address(this), _tokenId) - total, "BB: too much");
        } else {
            // yieldBox.transfer(from, address(this), _tokenId, share);
            bool isErr = pearlmit.transferFromERC1155(from, address(this), address(yieldBox), _tokenId, share);
            if (isErr) {
                revert TransferFailed();
            }
        }
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLCommon.sol#L165-L177

```solidity
    function _addTokens(address from, address, uint256 _assetId, uint256 share, uint256 total, bool skim) internal {
        if (skim) {
>>          if (share > yieldBox.balanceOf(address(this), _assetId) - total) {
                revert TooMuch();
            }
        } else {
            // yieldBox.transfer(from, address(this), _assetId, share);
            bool isErr = pearlmit.transferFromERC1155(from, address(this), address(yieldBox), _assetId, share);
            if (isErr) {
                revert TransferFailed();
            }
        }
    }
```

## Tool used

Manual Review

## Recommendation

Consider sending back (via `yieldBox.depositAsset`) the `memoryData.shareOut - memoryData.shareOwed` amount to the operating user.