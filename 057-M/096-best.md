Vast Bronze Salamander

medium

# SGL.borrow() the value of assets borrowed  greater than the value of debts recorded

## Summary
The current implementation of `SGL.borrow()` may result in the value of assets borrowed greater than the value of debts recorded, potentially leading to more bad debts.
## Vulnerability Detail
The implementation of `SGL.borrow()` : 

```solidity
contract SGLLendingCommon is SGLCommon {

    function _borrow(address from, address to, uint256 amount) internal returns (uint256 part, uint256 share) {
@>      share = yieldBox.toShare(assetId, amount, true);
        Rebase memory _totalAsset = totalAsset;
        if (_totalAsset.base < 1000) revert MinLimit();

        uint256 fullAssetAmountBefore = yieldBox.toAmount(assetId, _totalAsset.elastic, false) + totalBorrow.elastic;

        _totalAsset.elastic -= share.toUint128();

        uint256 feeAmount = (amount * borrowOpeningFee) / FEE_PRECISION; // A flat % fee is charged for any borrow

@>      (totalBorrow, part) = totalBorrow.add(amount + feeAmount, true);
        if (totalBorrowCap != 0) {
            if (totalBorrow.elastic > totalBorrowCap) revert BorrowCapReached();
        }
        userBorrowPart[from] += part;
        emit LogBorrow(from, to, amount, feeAmount, part);

        if (feeAmount > 0) {
            uint256 feeFraction = (feeAmount * _totalAsset.base) / fullAssetAmountBefore;
            _totalAsset.base += feeFraction.toUint128();
            balanceOf[address(penrose)] += feeFraction;
        }

        totalAsset = _totalAsset;

@>      yieldBox.transfer(address(this), to, assetId, share);
    }

```
The above code performs the following main steps:
1. `share = yieldBox.toShare(assetId, amount, true);`   ------->rounded up
2. `(totalBorrow, part) = totalBorrow.add(amount + feeAmount, true);`
3. `userBorrowPart[from] += part;`
4. `yieldBox.transfer(address(this), to, assetId, share);`

Let's assume that the yieldBox conversion ratio is 1:2, and the borrowed amount  = 1 (assuming it's small enough that fees are rounded down 0).
1. shares = yieldBox.toShare(1)= 1  ----->rounded up
2. `part = 1`
3. `userBorrowPart[from] += 1`
4. The user receives 1 share yieldBox.

However, 1 share yieldBox is worth 2 assets, while the recorded debt only old debt amount = 1.

In summary, shares are rounded up, the debt accounting still uses the original input amount, leading to borrowed shares having a value greater than the actual debt.

The example provided uses small values for explanatory purposes, but even with larger values, the same conclusion holds due to the presence of rounding up.

A similar issue exists in Morpho; see this Twitter thread for more details:https://twitter.com/windowhan/status/1759626078400970938

## Impact

Borrowed shares having a value greater than the actual debt could result in more bad debts. 



## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLendingCommon.sol#L62

## Tool used

Manual Review

## Recommendation

It is recommended to recalculate the debt based on shares to ensure that the recorded debt aligns with the actual value borrowed.

```diff
    function _borrow(address from, address to, uint256 amount) internal returns (uint256 part, uint256 share) {
        share = yieldBox.toShare(assetId, amount, true);
+       amount = yieldBox.toAmount(assetId, share, true);
```
