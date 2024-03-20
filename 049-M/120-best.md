Skinny Wool Mallard

medium

# Not properly tracking debt accrual leads mintOpenInterestDebt() to lose twTap rewards

## Summary

Debt accrual is tracked wrongly, making the expected twTap rewards to be potentially lost.

## Vulnerability Detail

Penrose’s `mintOpenInterestDebt()` function allows USDO to be minted and distributed as a reward to twTap holders based on the current USDO open interest.

In order to mint and distribute rewards, `mintOpenInterestDebt()` will perform the following steps:

1. Query the current `USDO.supply()`
2. Compute the total debt from all the markets (Origins included)
3. If `totalUsdoDebt > usdoSupply`, then distribute the difference among the twTap holders

```solidity
function mintOpenInterestDebt(address twTap) external onlyOwner { 
        uint256 usdoSupply = usdoToken.totalSupply();

        // nothing to mint when there's no activity
        if (usdoSupply > 0) {  
            // re-compute latest debt
            uint256 totalUsdoDebt = computeTotalDebt();  
   
            //add Origins debt 
            //Origins market doesn't accrue in time but increases totalSupply
            //and needs to be taken into account here
            uint256 len = allOriginsMarkets.length;
            for (uint256 i; i < len; i++) {
                IMarket market = IMarket(allOriginsMarkets[i]);
                if (isOriginRegistered[address(market)]) {
                    (uint256 elastic,) = market.totalBorrow();
                    totalUsdoDebt += elastic;
                }
            }
 
            //debt should always be > USDO supply
            if (totalUsdoDebt > usdoSupply) { 
                uint256 _amount = totalUsdoDebt - usdoSupply;

                //mint against the open interest; supply should be fully minted now
                IUsdo(address(usdoToken)).mint(address(this), _amount);

                //send it to twTap
                uint256 rewardTokenId = ITwTap(twTap).rewardTokenIndex(address(usdoToken));
                _distributeOnTwTap(_amount, rewardTokenId, address(usdoToken), ITwTap(twTap));
            }
        } 
    }
```

This approach has two main issues that make the current reward distribution malfunction:

1. Because debt is not actually tracked and is instead directly queried from the current total borrows via `computeTotalDebt()`, if users repay their debt prior to a reward distribution this debt won’t be considered for the fees, given that fees will always be calculated considering the current `totalUsdoDebt` and `usdoSupply`.
2. Bridging USDO is not considered
    1. If USDO is bridged from another chain to the current chain, then the `usdoToken.totalSupply()` will increment but the `totalUsdoDebt()` won’t. This will make rewards never be distributed because `usdoSupply` will always be greater than `totalUsdoDebt`.
    2. On the other hand, if USDO is bridged from the current chain to another chain, the `usdoToken.totalSupply()` will decrement and tokens will be burnt, while `totalUsdoDebt()` will remain the same. This will make more rewards than the expected ones to be distributed because `usdoSupply` will be way smaller than `totalUsdoDebt`.

## Proof of concept

Consider the following scenario: 1000 USDO are borrowed, and already 50 USDO have been accrued as debt. 

This makes USDO’s totalSupply() to be 1000, while `totalUsdoDebt` be 1050 USDO. If `mintOpenInterestDebt()` is called, 50 USDO should be minted and distributed among twTap holders.

However, prior to executing `mintOpenInterestDebt()`, a user bridges 100 USDO from chain B, making the total supply increment from 1000 USDO to 1100 USDO. Now, totalSupply() is 1100 USDO, while `totalUsdoDebt` is still 1050, making rewards not be distributed among users because `totalUsdoDebt < usdoSupply`.

## Impact

Medium. The fees to be distributed in twTap are likely to always be wrong, making one of the core governance functionalities (locking TAP in order to participate in Tapioca’s governance) be broken given that fee distributions (and thus the incentives to participate in governance) won’t be correct.

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/Penrose.sol#L263-L295

## Tool used

Manual Review

## Recommendation

One of the possible fixes for this issue is to track debt with a storage variable. Every time a repay is performed, the difference between elastic and base could be accrued to the variable, and such variable could be decremented when the fees distributions are performed. This makes it easier to compute the actual rewards and mitigates the cross-chain issue.
