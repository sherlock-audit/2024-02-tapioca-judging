Zealous Pineapple Duck

high

# Allowances is double spent in BBLeverage's and SGLLeverage's `sellCollateral()`

## Summary

Both `sellCollateral()` functions remove allowance twice, before and after target debt size control.

This way up to double allowance is removed each time: exactly double amount when repay amount is at or below the actual debt, somewhat less than double amount when repay amount is bigger than actual debt.

## Vulnerability Detail

Both `_allowedBorrow()` and `_repay()` operations spend the allowance from the caller. In the `sellCollateral()` case it should be done once, before the operation, since collateral removal is unconditional. It is spend twice, on collateral removal, and then on debt repayment now instead.

## Impact

BBLeverage's and SGLLeverage's `sellCollateral()` callers lose the approximately double amount of collateral allowance on each call. The operations with correctly set allowance amounts will be denied.

There are no prerequisites, so the probability is high. As the allowances are material and extra amounts can be directly exploitted via collateral removal (i.e. any extra allowance can be instantly turned to the same amount of collateral as long as borrower's account is healthy enough), so having allowances lost is somewhat lower/equivalent severity to loss of funds, i.e. have medium/high severity.

Likelihood: High + Impact: Medium/High = Severity: High.

## Code Snippet

The allowance is double written off in BBLeverage's and SGLLeverage's `sellCollateral()`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L126-L162

```solidity
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
>>      _allowedBorrow(from, share);
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
        yieldBox.depositAsset(collateralId, address(this), address(this), 0, memoryData.shareOut); // TODO Check for rounding attack?
        address(asset).safeApprove(address(yieldBox), 0);

        memoryData.partOwed = userBorrowPart[from];
        memoryData.amountOwed = totalBorrow.toElastic(memoryData.partOwed, true);
        memoryData.shareOwed = yieldBox.toShare(assetId, memoryData.amountOwed, true);
        if (memoryData.shareOwed <= memoryData.shareOut) {
>>          _repay(from, from, memoryData.partOwed);
        } else {
            //repay as much as we can
            uint256 partOut = totalBorrow.toBase(amountOut, false);
>>          _repay(from, from, partOut);
        }
    }
```



https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLeverage.sol#L106-L148

```solidity
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
        // Stack too deep fix
        _SellCollateralCalldata memory calldata_;
        {
            calldata_.from = from;
            calldata_.share = share;
            calldata_.data = data;
        }

>>      _allowedBorrow(calldata_.from, calldata_.share);
        _removeCollateral(calldata_.from, address(this), calldata_.share);

        yieldBox.withdraw(collateralId, address(this), address(leverageExecutor), 0, calldata_.share);
        uint256 leverageAmount = yieldBox.toAmount(collateralId, calldata_.share, false);
        amountOut = leverageExecutor.getAsset(
            assetId, address(collateral), address(asset), leverageAmount, calldata_.from, calldata_.data
        );
        uint256 shareOut = yieldBox.toShare(assetId, amountOut, false);

        address(asset).safeApprove(address(yieldBox), type(uint256).max);
        yieldBox.depositAsset(assetId, address(this), address(this), 0, shareOut);
        address(asset).safeApprove(address(yieldBox), 0);

        uint256 partOwed = userBorrowPart[calldata_.from];
        uint256 amountOwed = totalBorrow.toElastic(partOwed, true);
        uint256 shareOwed = yieldBox.toShare(assetId, amountOwed, true);
        if (shareOwed <= shareOut) {
>>          _repay(calldata_.from, calldata_.from, false, partOwed);
        } else {
            //repay as much as we can
            uint256 partOut = totalBorrow.toBase(amountOut, false);
>>          _repay(calldata_.from, calldata_.from, false, partOut);
        }
    }
```

## Tool used

Manual Review

## Recommendation

Consider adding a flag to `_repay()`, indicating that allowance spending was already recorded.

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLendingCommon.sol#L107-L122

```diff
-   function _repay(address from, address to, uint256 part) internal returns (uint256 amount) {
+   function _repay(address from, address to, uint256 part, bool checkAllowance) internal returns (uint256 amount) {
        if (part > userBorrowPart[to]) {
            part = userBorrowPart[to];
        }
        if (part == 0) revert NothingToRepay();

        // @dev check allowance
-       if (msg.sender != from) {
+       if (checkAllowance && msg.sender != from) {
            uint256 partInAmount;
            Rebase memory _totalBorrow = totalBorrow;
            (_totalBorrow, partInAmount) = _totalBorrow.sub(part, false);
            uint256 allowanceShare =
                _computeAllowanceAmountInAsset(to, exchangeRate, partInAmount, _safeDecimals(asset));
            if (allowanceShare == 0) revert AllowanceNotValid();
            _allowedBorrow(from, allowanceShare);
        }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLendingCommon.sol#L91-L106

```diff
-   function _repay(address from, address to, bool skim, uint256 part) internal returns (uint256 amount) {
+   function _repay(address from, address to, bool skim, uint256 part, bool checkAllowance) internal returns (uint256 amount) {
        if (part > userBorrowPart[to]) {
            part = userBorrowPart[to];
        }
        if (part == 0) revert NothingToRepay();

-       if (msg.sender != from) {
+       if (checkAllowance && msg.sender != from) {
            uint256 partInAmount;
            Rebase memory _totalBorrow = totalBorrow;
            (_totalBorrow, partInAmount) = _totalBorrow.sub(part, false);

            uint256 allowanceShare =
                _computeAllowanceAmountInAsset(to, exchangeRate, partInAmount, _safeDecimals(asset));
            if (allowanceShare == 0) revert AllowanceNotValid();
            _allowedBorrow(from, allowanceShare);
        }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L126-L162

```diff
    function sellCollateral(address from, uint256 share, bytes calldata data)
        ...
    {
        ...
        _allowedBorrow(from, share);
        _removeCollateral(from, address(this), share);

        ...

        memoryData.partOwed = userBorrowPart[from];
        memoryData.amountOwed = totalBorrow.toElastic(memoryData.partOwed, true);
        memoryData.shareOwed = yieldBox.toShare(assetId, memoryData.amountOwed, true);
        if (memoryData.shareOwed <= memoryData.shareOut) {
-           _repay(from, from, memoryData.partOwed);
+           _repay(from, from, memoryData.partOwed, false);
        } else {
            //repay as much as we can
            uint256 partOut = totalBorrow.toBase(amountOut, false);
-           _repay(from, from, partOut);
+           _repay(from, from, partOut, false);
        }
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLeverage.sol#L106-L148

```diff
    function sellCollateral(address from, uint256 share, bytes calldata data)
        ...
    {
        ...

        _allowedBorrow(calldata_.from, calldata_.share);
        _removeCollateral(calldata_.from, address(this), calldata_.share);

        ...

        uint256 partOwed = userBorrowPart[calldata_.from];
        uint256 amountOwed = totalBorrow.toElastic(partOwed, true);
        uint256 shareOwed = yieldBox.toShare(assetId, amountOwed, true);
        if (shareOwed <= shareOut) {
-           _repay(calldata_.from, calldata_.from, false, partOwed);
+           _repay(calldata_.from, calldata_.from, false, partOwed, false);
        } else {
            //repay as much as we can
            uint256 partOut = totalBorrow.toBase(amountOut, false);
-           _repay(calldata_.from, calldata_.from, false, partOut);
+           _repay(calldata_.from, calldata_.from, false, partOut, false);
        }
    }
```

All other `_repay()` calls should by default use `checkAllowance == true`.