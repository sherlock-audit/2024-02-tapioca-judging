Vast Bronze Salamander

high

# Unupdated totalBorrow After BigBang Liquidation

## Summary
During the liquidation process, BigBang only reduces the user's `userBorrowPart[user]`, but fails to update the global `totalBorrow`. Consequently, all subsequent debt calculations are incorrect.

## Vulnerability Detail
Currently, the implementation relies on the `BBLiquidation._updateBorrowAndCollateralShare()` method to calculate user debt repayment and collateral collection. 
The code snippet is as follows:
```solidity
    function _liquidateUser(
        address user,
        uint256 maxBorrowPart,
        IMarketLiquidatorReceiver _liquidatorReceiver,
        bytes calldata _liquidatorReceiverData,
        uint256 _exchangeRate,
        uint256 minLiquidationBonus
    ) private {
        uint256 callerReward = _getCallerReward(user, _exchangeRate);

        (uint256 borrowAmount,, uint256 collateralShare) =
@>          _updateBorrowAndCollateralShare(user, maxBorrowPart, minLiquidationBonus, _exchangeRate);
@>      totalCollateralShare = totalCollateralShare > collateralShare ? totalCollateralShare - collateralShare : 0;
        uint256 borrowShare = yieldBox.toShare(assetId, borrowAmount, true);

        (uint256 returnedShare,) =
            _swapCollateralWithAsset(collateralShare, _liquidatorReceiver, _liquidatorReceiverData);
        if (returnedShare < borrowShare) revert AmountNotValid();

        (uint256 feeShare, uint256 callerShare) = _extractLiquidationFees(returnedShare, borrowShare, callerReward);

        IUsdo(address(asset)).burn(address(this), borrowAmount);

        address[] memory _users = new address[](1);
        _users[0] = user;
        emit Liquidated(msg.sender, _users, callerShare, feeShare, borrowAmount, collateralShare);
    }

    function _updateBorrowAndCollateralShare(
        address user,
        uint256 maxBorrowPart,
        uint256 minLiquidationBonus, // min liquidation bonus to accept (default 0)
        uint256 _exchangeRate
    ) private returns (uint256 borrowAmount, uint256 borrowPart, uint256 collateralShare) {
        if (_exchangeRate == 0) revert ExchangeRateNotValid();

        // get collateral amount in asset's value
        uint256 collateralPartInAsset = (
            yieldBox.toAmount(collateralId, userCollateralShare[user], false) * EXCHANGE_RATE_PRECISION
        ) / _exchangeRate;

        // compute closing factor (liquidatable amount)
        uint256 borrowPartWithBonus =
            computeClosingFactor(userBorrowPart[user], collateralPartInAsset, FEE_PRECISION_DECIMALS);

        // limit liquidable amount before bonus to the current debt
        uint256 userTotalBorrowAmount = totalBorrow.toElastic(userBorrowPart[user], true);
        borrowPartWithBonus = borrowPartWithBonus > userTotalBorrowAmount ? userTotalBorrowAmount : borrowPartWithBonus;

        // check the amount to be repaid versus liquidator supplied limit
        borrowPartWithBonus = borrowPartWithBonus > maxBorrowPart ? maxBorrowPart : borrowPartWithBonus;
        borrowAmount = borrowPartWithBonus;

        // compute part units, preventing rounding dust when liquidation is full
        borrowPart = borrowAmount == userTotalBorrowAmount
            ? userBorrowPart[user]
            : totalBorrow.toBase(borrowPartWithBonus, false);
        if (borrowPart == 0) revert Solvent();

        if (liquidationBonusAmount > 0) {
            borrowPartWithBonus = borrowPartWithBonus + (borrowPartWithBonus * liquidationBonusAmount) / FEE_PRECISION;
        }

        if (collateralPartInAsset < borrowPartWithBonus) {
            if (collateralPartInAsset <= userTotalBorrowAmount) {
                revert BadDebt();
            }
            // If current debt is covered by collateral fully
            // then there is some liquidation bonus,
            // so liquidation can proceed if liquidator's minimum is met
            if (minLiquidationBonus > 0) {
                // `collateralPartInAsset > borrowAmount` as `borrowAmount <= userTotalBorrowAmount`
                uint256 effectiveBonus = ((collateralPartInAsset - borrowAmount) * FEE_PRECISION) / borrowAmount;
                if (effectiveBonus < minLiquidationBonus) {
                    revert InsufficientLiquidationBonus();
                }
                collateralShare = userCollateralShare[user];
            } else {
                revert InsufficientLiquidationBonus();
            }
        } else {
            collateralShare =
                yieldBox.toShare(collateralId, (borrowPartWithBonus * _exchangeRate) / EXCHANGE_RATE_PRECISION, false);
            if (collateralShare > userCollateralShare[user]) {
                revert NotEnoughCollateral();
            }
        }

@>      userBorrowPart[user] -= borrowPart;
@>      userCollateralShare[user] -= collateralShare;
    }


```
The methods mentioned above update the user-specific variables `userBorrowPart[user]` and `userCollateralShare[user]` within the `_updateBorrowAndCollateralShare()` method. 
Additionally, the global variable `totalCollateralShare` is updated within the `_liquidateUser()` method.

However, there's another crucial global variable, `totalBorrow`, which remains unaltered throughout the entire liquidation process.

Failure to update `totalBorrow` during liquidation will result in incorrect subsequent loan-related calculations.

Note: SGL Liquidation has the same issues

## Impact

The lack of an update to `totalBorrow` during liquidation leads to inaccuracies in subsequent loan-related calculations. 
For instance, this affects interest accumulation and the amount of interest due.

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLiquidation.sol#L218-L228
## Tool used

Manual Review

## Recommendation

```diff
    function _liquidateUser(
        address user,
        uint256 maxBorrowPart,
        IMarketLiquidatorReceiver _liquidatorReceiver,
        bytes calldata _liquidatorReceiverData,
        uint256 _exchangeRate,
        uint256 minLiquidationBonus
    ) private {
        uint256 callerReward = _getCallerReward(user, _exchangeRate);

-       (uint256 borrowAmount,, uint256 collateralShare) =
+       (uint256 borrowAmount,uint256 borrowPart, uint256 collateralShare) =
            _updateBorrowAndCollateralShare(user, maxBorrowPart, minLiquidationBonus, _exchangeRate);
        totalCollateralShare = totalCollateralShare > collateralShare ? totalCollateralShare - collateralShare : 0;
+       totalBorrow.elastic -= borrowAmount.toUint128();
+       totalBorrow.base -= borrowPart.toUint128();
```
