Modern Mandarin Wasp

medium

# BBCommon::_accrue wrong value is used to prevent overflow

## Summary
A mechanism is used in `_accrue`, to prevent overflow in order to avoid Dos on multiple entrypoints of a BigBang market (many external functions call on `_accrue` before executing their logic). However the wrong value is used to prevent an overflow, and even though it could be prevented the first time, it should overflow the second one it is called

## Vulnerability Detail
We can see that the value to be accrued is clamped to [type(uint128).max - totalBorrowCap](https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBCommon.sol#L107)

Which works the first time to avoid an overflow since `totalBorrow.elastic` should be less than `totalBorrowCap`. However if `totalBorrow.elastic` is already bigger than `totalBorrowCap` (due to a previous accrual), this clamping does not prevent overflow.

## Impact
`_accrue` can still revert due to an overflow blocking most of the functions of a `BigBang` market.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Clamp accrued value to `type(uint128).max - totalBorrow.elastic` instead