Zealous Pineapple Duck

high

# BBLeverage's and SGLLeverage's `buyCollateral()` remove the required funds from the target twice

## Summary

The collateral purchase proceedings of leverage buy operation aren't returned to the user, but kept with the contract instead, so the funds are being requested from the user twice, first in a usual borrow and buy workflow (user now had a liability of `collateralShare` size), then once again via `_addCollateral` (user has transferred `collateralShare` directly in addition to that).

## Vulnerability Detail

Both `buyCollateral()` functions are now broken this way, removing twice the value from the user. The accounting is being updated accordingly to reflect only one instance, so the funds are lost for the user.

I.e. after each of the operations user borrowed the `collateralShare` worth of `asset` and then additionally to that supplied it directly via `_addCollateral`. As only one collateral addition is recorded, the loss for them is `collateralShare` of collateral. Since the accounting is updated it will not be possible to manually fix the situation in production, so the redeployment would be due.

## Impact

It's an unconditional user loss each time the operations are used. There are no prerequisites, so the probability of it is high. Funds loss impact has high severity.

Likelihood: High + Impact: High = Severity: Critical/High.

## Code Snippet

First `calldata_.borrowAmount` is being borrowed for `calldata_.from`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L82-L102

```solidity
        {
            (, uint256 borrowShare) = _borrow(
>>              calldata_.from,
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
                calldata_.from,
                calldata_.data
            );
        }
        uint256 collateralShare = yieldBox.toShare(collateralId, amountOut, false);
```

And while borrow proceedings are being held in the contract, the same funds are being requested again from `calldata_.from` in the collateral form:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L102-L110

```solidity
        uint256 collateralShare = yieldBox.toShare(collateralId, amountOut, false);
        address(asset).safeApprove(address(yieldBox), type(uint256).max);
        yieldBox.depositAsset(collateralId, address(this), address(this), 0, collateralShare); // TODO Check for rounding attack?
        address(asset).safeApprove(address(yieldBox), 0);

        if (collateralShare == 0) revert CollateralShareNotValid();
        _allowedBorrow(calldata_.from, collateralShare);
>>      _addCollateral(calldata_.from, calldata_.from, false, 0, collateralShare);
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLendingCommon.sol#L40-L49

```solidity
    function _addCollateral(address from, address to, bool skim, uint256 amount, uint256 share) internal {
        if (share == 0) {
            share = yieldBox.toShare(collateralId, amount, false);
        }
        userCollateralShare[to] += share;
        uint256 oldTotalCollateralShare = totalCollateralShare;
        totalCollateralShare = oldTotalCollateralShare + share;
>>      _addTokens(from, collateralId, share, oldTotalCollateralShare, skim);
        emit LogAddCollateral(skim ? address(yieldBox) : from, to, share);
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBCommon.sol#L128-L138

```solidity
    function _addTokens(address from, uint256 _tokenId, uint256 share, uint256 total, bool skim) internal {
        if (skim) {
            require(share <= yieldBox.balanceOf(address(this), _tokenId) - total, "BB: too much");
        } else {
            // yieldBox.transfer(from, address(this), _tokenId, share);
>>          bool isErr = pearlmit.transferFromERC1155(from, address(this), address(yieldBox), _tokenId, share);
            if (isErr) {
                revert TransferFailed();
            }
        }
    }
```

This last step of requesting `collateralShare` that is already with BBLeverage/SGLLeverage contract doesn't make sense for `buyCollateral()`, being a leftover from the legacy logic (when the collateral funds were transferred to a user, while now they aren't).

SGLLeverage situation is the same, first borrow `calldata_.borrowAmount`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLeverage.sol#L73-L85

```solidity
>>      (, uint256 borrowShare) = _borrow(calldata_.from, address(this), calldata_.borrowAmount);

        (uint256 borrowShareToAmount,) =
            yieldBox.withdraw(assetId, address(this), address(leverageExecutor), 0, borrowShare);
        amountOut = leverageExecutor.getCollateral(
            collateralId,
            address(asset),
            address(collateral),
            supplyShareToAmount + borrowShareToAmount,
            calldata_.from,
            calldata_.data
        );
        uint256 collateralShare = yieldBox.toShare(collateralId, amountOut, false);
```

Then additionally request `collateralShare` from `calldata_.from`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLeverage.sol#L85-L93

```solidity
        uint256 collateralShare = yieldBox.toShare(collateralId, amountOut, false);
        if (collateralShare == 0) revert CollateralShareNotValid();
        address(asset).safeApprove(address(yieldBox), type(uint256).max);
        yieldBox.depositAsset(collateralId, address(this), address(this), 0, collateralShare); // TODO Check for rounding attack?
        address(asset).safeApprove(address(yieldBox), 0);

        _allowedBorrow(calldata_.from, collateralShare);
>>      _addCollateral(calldata_.from, calldata_.from, false, 0, collateralShare);
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLendingCommon.sol#L39-L50

```solidity
    function _addCollateral(address from, address to, bool skim, uint256 amount, uint256 share) internal {
        if (share == 0) {
            share = yieldBox.toShare(collateralId, amount, false);
        }
        uint256 oldTotalCollateralShare = totalCollateralShare;
        userCollateralShare[to] += share;
        totalCollateralShare = oldTotalCollateralShare + share;

>>      _addTokens(from, to, collateralId, share, oldTotalCollateralShare, skim);

        emit LogAddCollateral(skim ? address(yieldBox) : from, to, share);
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLCommon.sol#L165-L177

```solidity
    function _addTokens(address from, address, uint256 _assetId, uint256 share, uint256 total, bool skim) internal {
        if (skim) {
            if (share > yieldBox.balanceOf(address(this), _assetId) - total) {
                revert TooMuch();
            }
        } else {
            // yieldBox.transfer(from, address(this), _assetId, share);
>>          bool isErr = pearlmit.transferFromERC1155(from, address(this), address(yieldBox), _assetId, share);
            if (isErr) {
                revert TransferFailed();
            }
        }
    }
```

The root cause is that leverageExecutor's logic has changed: previously `getCollateral()` put the funds to `from`, while now the funds are directly transferred to the BBLeverage/SGLLeverage contract, which is `msg.sender` for `leverageExecutor`, e.g.:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/leverage/SimpleLeverageExecutor.sol#L38-L47

```solidity
    function getCollateral(
        ....
    ) external payable override returns (uint256 collateralAmountOut) {
        // Should be called only by approved SGL/BB markets.
        if (!cluster.isWhitelisted(0, msg.sender)) revert SenderNotValid();
>>      return _swapAndTransferToSender(true, assetAddress, collateralAddress, assetAmountIn, swapperData);
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/leverage/BaseLeverageExecutor.sol#L129-L159

```solidity
    function _swapAndTransferToSender(
        bool sendBack,
        ...
    ) internal returns (uint256 amountOut) {
        ...

        // If the tokenOut is a tOFT, wrap it. Handles ETH and ERC20.
        // If `sendBack` is true, wrap the `amountOut to` the sender. else, wrap it to this contract.
        if (swapData.toftInfo.isTokenOutToft) {
            _handleToftWrapToSender(sendBack, tokenOut, amountOut);
        } else if (sendBack == true) {
            // If the token wasn't sent by the wrap OP, send it as a transfer.
>>          IERC20(tokenOut).safeTransfer(msg.sender, amountOut);
        }
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/leverage/AssetTotsDaiLeverageExecutor.sol#L38-L65

```solidity
    function getCollateral(address assetAddress, address collateralAddress, uint256 assetAmountIn, bytes calldata data)
        ...
    {
        ...

        // Wrap into tsDai to sender
        sDaiAddress.safeApprove(collateralAddress, collateralAmountOut);
>>      ITOFT(collateralAddress).wrap(address(this), msg.sender, collateralAmountOut);
        sDaiAddress.safeApprove(collateralAddress, 0);
    }
```

## Tool used

Manual Review

## Recommendation

Consider adding a flag to `_addCollateral()`, indicating that the funds were already transferred to the contract and only accounting update is needed, e.g.:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLendingCommon.sol#L40-L49

```diff
-   function _addCollateral(address from, address to, bool skim, uint256 amount, uint256 share) internal {
+   function _addCollateral(address from, address to, bool skim, uint256 amount, uint256 share, bool addTokens) internal {
        if (share == 0) {
            share = yieldBox.toShare(collateralId, amount, false);
        }
        userCollateralShare[to] += share;
        uint256 oldTotalCollateralShare = totalCollateralShare;
        totalCollateralShare = oldTotalCollateralShare + share;
-       _addTokens(from, collateralId, share, oldTotalCollateralShare, skim);
+       if (addTokens) _addTokens(from, collateralId, share, oldTotalCollateralShare, skim);
        emit LogAddCollateral(skim ? address(yieldBox) : from, to, share);
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLendingCommon.sol#L39-L50

```diff
-   function _addCollateral(address from, address to, bool skim, uint256 amount, uint256 share) internal {
+   function _addCollateral(address from, address to, bool skim, uint256 amount, uint256 share, bool addTokens) internal {
        if (share == 0) {
            share = yieldBox.toShare(collateralId, amount, false);
        }
        uint256 oldTotalCollateralShare = totalCollateralShare;
        userCollateralShare[to] += share;
        totalCollateralShare = oldTotalCollateralShare + share;

-       _addTokens(from, to, collateralId, share, oldTotalCollateralShare, skim);
+       if (addTokens) _addTokens(from, to, collateralId, share, oldTotalCollateralShare, skim);

        emit LogAddCollateral(skim ? address(yieldBox) : from, to, share);
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L102-L110

```diff
        uint256 collateralShare = yieldBox.toShare(collateralId, amountOut, false);
        address(asset).safeApprove(address(yieldBox), type(uint256).max);
        yieldBox.depositAsset(collateralId, address(this), address(this), 0, collateralShare); // TODO Check for rounding attack?
        address(asset).safeApprove(address(yieldBox), 0);

        if (collateralShare == 0) revert CollateralShareNotValid();
        _allowedBorrow(calldata_.from, collateralShare);
-       _addCollateral(calldata_.from, calldata_.from, false, 0, collateralShare);
+       _addCollateral(calldata_.from, calldata_.from, false, 0, collateralShare, false);
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLeverage.sol#L85-L93

```diff
        uint256 collateralShare = yieldBox.toShare(collateralId, amountOut, false);
        if (collateralShare == 0) revert CollateralShareNotValid();
        address(asset).safeApprove(address(yieldBox), type(uint256).max);
        yieldBox.depositAsset(collateralId, address(this), address(this), 0, collateralShare); // TODO Check for rounding attack?
        address(asset).safeApprove(address(yieldBox), 0);

        _allowedBorrow(calldata_.from, collateralShare);
-       _addCollateral(calldata_.from, calldata_.from, false, 0, collateralShare);
+       _addCollateral(calldata_.from, calldata_.from, false, 0, collateralShare, false);
    }
```

All other `_addCollateral()` instances should be updated to go with `addTokens == true`.