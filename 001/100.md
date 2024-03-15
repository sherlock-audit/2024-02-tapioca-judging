Vast Bronze Salamander

medium

# buyCollateral()  does not work properly

## Summary
The `BBLeverage.buyCollateral()` function does not work as expected.

## Vulnerability Detail
The implementation of `BBLeverage.buyCollateral()` is as follows:

```solidity
    function buyCollateral(address from, uint256 borrowAmount, uint256 supplyAmount, bytes calldata data)
        external
        optionNotPaused(PauseType.LeverageBuy)
        solvent(from, false)
        notSelf(from)
        returns (uint256 amountOut)
    {
        if (address(leverageExecutor) == address(0)) {
            revert LeverageExecutorNotValid();
        }

        // Stack too deep fix
        _BuyCollateralCalldata memory calldata_;
        _BuyCollateralMemoryData memory memoryData;
        {
            calldata_.from = from;
            calldata_.borrowAmount = borrowAmount;
            calldata_.supplyAmount = supplyAmount;
            calldata_.data = data;
        }

        {
            uint256 supplyShare = yieldBox.toShare(assetId, calldata_.supplyAmount, true);
            if (supplyShare > 0) {
                (memoryData.supplyShareToAmount,) =
                    yieldBox.withdraw(assetId, calldata_.from, address(leverageExecutor), 0, supplyShare);
            }
        }

        {
            (, uint256 borrowShare) = _borrow(
                calldata_.from,
                address(this),
                calldata_.borrowAmount,
                _computeVariableOpeningFee(calldata_.borrowAmount)
            );
            (memoryData.borrowShareToAmount,) =
                yieldBox.withdraw(assetId, address(this), address(leverageExecutor), 0, borrowShare);
        }
        {
            amountOut = leverageExecutor.getCollateral(
                collateralId,
                address(asset),
                address(collateral),
                memoryData.supplyShareToAmount + memoryData.borrowShareToAmount,
@>              calldata_.from,
                calldata_.data
            );
        }
        uint256 collateralShare = yieldBox.toShare(collateralId, amountOut, false);
@>      address(asset).safeApprove(address(yieldBox), type(uint256).max);
@>      yieldBox.depositAsset(collateralId, address(this), address(this), 0, collateralShare); // TODO Check for rounding attack?
@>      address(asset).safeApprove(address(yieldBox), 0);

        if (collateralShare == 0) revert CollateralShareNotValid();
        _allowedBorrow(calldata_.from, collateralShare);
        _addCollateral(calldata_.from, calldata_.from, false, 0, collateralShare);
    }
```

The code above has several issues:
1. `leverageExecutor.getCollateral()` receiver should be `address(this)`.  ---> for 2th step deposit to YB
2. `address(asset).safeApprove()` should use `address(collateral).safeApprove()`.  
3. `yieldBox.depositAsset()` receiver should be `calldata_.from`.  ----> for next execute _addCollateral(calldata_.from)


Note: SGLLeverage.sol have same issue

## Impact
`buyCollateral()` does not work properly.


## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L53C1-L110C6
## Tool used

Manual Review

## Recommendation

```diff
    function buyCollateral(address from, uint256 borrowAmount, uint256 supplyAmount, bytes calldata data)
        external
        optionNotPaused(PauseType.LeverageBuy)
        solvent(from, false)
        notSelf(from)
        returns (uint256 amountOut)
    {
....

        {
            (, uint256 borrowShare) = _borrow(
                calldata_.from,
                address(this),
                calldata_.borrowAmount,
                _computeVariableOpeningFee(calldata_.borrowAmount)
            );
            (memoryData.borrowShareToAmount,) =
                yieldBox.withdraw(assetId, address(this), address(leverageExecutor), 0, borrowShare);
        }
        {
            amountOut = leverageExecutor.getCollateral(
                collateralId,
                address(asset),
                address(collateral),
                memoryData.supplyShareToAmount + memoryData.borrowShareToAmount,
-               calldata_.from,
+               address(this),
                calldata_.data
            );
        }
        uint256 collateralShare = yieldBox.toShare(collateralId, amountOut, false);
-       address(asset).safeApprove(address(yieldBox), type(uint256).max);
-       yieldBox.depositAsset(collateralId, address(this), address(this), 0, collateralShare); // TODO Check for rounding attack?
-       address(asset).safeApprove(address(yieldBox), 0);
+       address(collateral).safeApprove(address(yieldBox), type(uint256).max);
+       yieldBox.depositAsset(collateralId, address(this), calldata_.from, 0, collateralShare);
+       address(collateral).safeApprove(address(yieldBox), 0);

        if (collateralShare == 0) revert CollateralShareNotValid();
        _allowedBorrow(calldata_.from, collateralShare);
        _addCollateral(calldata_.from, calldata_.from, false, 0, collateralShare);
    }
```
