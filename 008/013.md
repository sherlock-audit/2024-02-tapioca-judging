Round Pearl Crow

high

# Malicious user can borrow and leave protocol in bad debt in SGLBorrow.sol

## Summary
`from` isn't checked to be solvent after borrow.
## Vulnerability Detail
when borrowing the user who is `from` is checked to ensure he is solvent  before he can borrow.

But the issue is that the check isn't redone after the user borrows. 

So a user can borrow an amount that is bigger than his collateral, so even after he is liquidated his collateral won't be able to cover the amount he borrowed, leaving the protocol in bad Debt.

A more vivid scenario:
- user has supplied 1000$ worth of collateral, so when he calls borrow() he is solvent 
- now we can't trust user to borrow maybe 800$ worth of `shares`.  He could borrow up to 5000$ worth of shares and become insolvent immediately. and the borrow() will still go through. 

Now even if he is liquidated 4000$ worth of shares is  unsettled and lost
## Impact
 Malicious user can borrow and leave protocol in bad debt in SGLBorrow.sol
## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLBorrow.sol#L29-L46
## Tool used

Manual Review

## Recommendation

`from` should be checked to be solvent after borrow.
```solidity
 function borrow(address from, address to, uint256 amount)// @audit-issue can borrow amount and become insolvent 
        external
        optionNotPaused(PauseType.Borrow)
        solvent(from, false)
        notSelf(to)
        returns (uint256 part, uint256 share)
    {
        if (amount == 0) return (0, 0);
        uint256 feeAmount = (amount * borrowOpeningFee) / FEE_PRECISION;
        uint256 allowanceShare =
            _computeAllowanceAmountInAsset(from, exchangeRate, amount + feeAmount, _safeDecimals(asset));

        if (allowanceShare == 0) revert AllowanceNotValid();

        _allowedBorrow(from, allowanceShare);

        (part, share) = _borrow(from, to, amount);

+>>>    require(_isSolvent(from, exchangeRate, false), "Market: insolvent");
    }
```