Modern Mandarin Wasp

medium

# Penrose::_depositFeesToTwTap can unexpectedly revert due to amount rounded down

## Summary
_depositFeesToTwTap computes the amount of fees withdrawn after having it withdrawn. There may be a difference with actual fees withdrawn which could cause the function to revert unexpectedly

## Vulnerability Detail
We can see the fees being withdrawn [here](https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/Penrose.sol#L573), but instead of using the amount withdrawn, the amount is recomputed from shares [here](https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/Penrose.sol#L576)

This can be problematic if during first call the fee amount is rounded down, but after being withdrawn, it is not being rounded down

### Example

- before withdrawal:

base: 7
elastic: 13
share: 2

amount withdrawn: 13*2/7 = 3
shares deducted = 2

- after withdrawal:

base: 5
elastic: 10
share: 2

amount computed: 10*2/5 = 4

The transfer reverts because only 3 has been withdrawn

## Impact
The call will withdraw unexpectedly in some cases, dosing fees withdrawal

## Code Snippet

## Tool used

Manual Review

## Recommendation
Compute the amount of fees which will be withdrawn before making the actual withdrawal in `_depositFeesToTwTap`