Zealous Pineapple Duck

high

# SGL and BB repay do not round up both on allowance spending and elastic amount

## Summary

The shares to amount translation isn't done in protocol favor on BB/SGL repay.

## Vulnerability Detail

Amount provided by the user can be less than shares being written off on debt repayment.

## Impact

Protocol can be exploited by paying out very little amount many times, when the absence of rounding up becomes material. The impact is up to closing the debt for free.

## Code Snippet

SGL repay do not round up both on allowance spending and elastic amount for the given base amount:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLendingCommon.sol#L91-L107

```solidity
    function _repay(address from, address to, bool skim, uint256 part) internal returns (uint256 amount) {
        if (part > userBorrowPart[to]) {
            part = userBorrowPart[to];
        }
        if (part == 0) revert NothingToRepay();

        if (msg.sender != from) {
            uint256 partInAmount;
            Rebase memory _totalBorrow = totalBorrow;
>>          (_totalBorrow, partInAmount) = _totalBorrow.sub(part, false);

            uint256 allowanceShare =
                _computeAllowanceAmountInAsset(to, exchangeRate, partInAmount, _safeDecimals(asset));
            if (allowanceShare == 0) revert AllowanceNotValid();
            _allowedBorrow(from, allowanceShare);
        }
>>      (totalBorrow, amount) = totalBorrow.sub(part, false);
```

BB repay correctly reduces allowance, but doesn't round up the elastic amount for the given base amount:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLendingCommon.sol#L107-L126

```solidity
    function _repay(address from, address to, uint256 part) internal returns (uint256 amount) {
        if (part > userBorrowPart[to]) {
            part = userBorrowPart[to];
        }
        if (part == 0) revert NothingToRepay();

        // @dev check allowance
        if (msg.sender != from) {
            uint256 partInAmount;
            Rebase memory _totalBorrow = totalBorrow;
>>          (_totalBorrow, partInAmount) = _totalBorrow.sub(part, false);
            uint256 allowanceShare =
                _computeAllowanceAmountInAsset(to, exchangeRate, partInAmount, _safeDecimals(asset));
            if (allowanceShare == 0) revert AllowanceNotValid();
            _allowedBorrow(from, allowanceShare);
        }

        // @dev sub `part` of totalBorrow
>>      (totalBorrow, amount) = totalBorrow.sub(part, true);
        userBorrowPart[to] -= part;
```

## Tool used

Manual Review

## Recommendation

Consider using `true` for amount calculations in all this cases, rounding the amounts up for the shares given.