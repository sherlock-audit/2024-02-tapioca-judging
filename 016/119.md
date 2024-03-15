Skinny Wool Mallard

medium

# Variable opening fee will always be wrongly computed if collateral is not a stablecoin

## Summary

Borrowing fees will be computed wrongly because of a combination of hardcoded values and a wrongly implemented setter function.

## Vulnerability Detail

Tapioca applies a linearly scaling creation fee to open a new CDP in Big Bang markets. This is done via the internal `_computeVariableOpeningFee()` function every time a new borrow is performed.

In order to compute the variable fee, the exchange rate will be queried. This rate is important in order to understand the current price of USDO related to the collateral asset. 

- If `_exchangeRate >= minMintFeeStart`, then `minMintFee` will be applied.
- If `_exchangeRate <= maxMintFeeStart`, then `maxMintFee` will be applied
- Otherwise, a proportional percentage will be applied to compue the fee

As per the comment in the code snippet shows below, Tapioca wrongly assumes that the exchange rate will always be `USDO <> USDC`, when in reality the actual collateral will dictate the exchange rate returned. 

It is also important to note the fact that contrary to what one would assume, `maxMintFeeStart` is assumed to be smaller than `minMintFeeStart` in order to perform the calculations:

```solidity
// BBLendingCommon.sol

function _computeVariableOpeningFee(uint256 amount) internal returns (uint256) {
        if (amount == 0) return 0; 
 
        //get asset <> USDC price ( USDO <> USDC ) 
        (bool updated, uint256 _exchangeRate) = assetOracle.get(oracleData); 
        if (!updated) revert OracleCallFailed();
   
        if (_exchangeRate >= minMintFeeStart) { 
            return (amount * minMintFee) / FEE_PRECISION;
        }
        if (_exchangeRate <= maxMintFeeStart) { 
            return (amount * maxMintFee) / FEE_PRECISION;
        }
     
        uint256 fee = maxMintFee
            - (((_exchangeRate - maxMintFeeStart) * (maxMintFee - minMintFee)) / (minMintFeeStart - maxMintFeeStart)); 
 
        if (fee > maxMintFee) return (amount * maxMintFee) / FEE_PRECISION;
        if (fee < minMintFee) return (amount * minMintFee) / FEE_PRECISION;

        if (fee > 0) {
            return (amount * fee) / FEE_PRECISION;
        }
        return 0;
    }
```

It is also important to note that `minMintFeeStart` and `maxMintFeeStart` are hardcoded when being initialized inside `BigBang.sol` (as mentioned, `maxMintFeeStart` is smaller than `minMintFeeStart`):

```solidity
// BigBang.sol

function _initCoreStorage(
        IPenrose _penrose,
        IERC20 _collateral,
        uint256 _collateralId,
        ITapiocaOracle _oracle,
        uint256 _exchangeRatePrecision,
        uint256 _collateralizationRate,
        uint256 _liquidationCollateralizationRate,
        ILeverageExecutor _leverageExecutor
    ) private {
        ...
        
        maxMintFeeStart = 975000000000000000; // 0.975 *1e18
        minMintFeeStart = 1000000000000000000; // 1*1e18

        ...
    } 
```

While the values hardcoded initially to values that are coherent for a USDO <> stablecoin exchange rate, these values won’t make sense if we find ourselves fetching an exchcange rate of an asset not stable.

Let’s say the collateral asset is ETH. If ETH is at 4000$, then the exchange rate will return a value of 0,00025. This will make the computation inside `_computeVariableOpeningFee()` always apply the maximum fee when borrowing because `_exchangeRate` is always smaller than `maxMintFeeStart` by default.

Although this has an easy fix (changing the values stored in `maxMintFeeStart` and  `minMintFeeStart`), this can’t be properly done because the `setMinAndMaxMintRange()` function wrongly assumes that `minMintFeeStart` must be smaller than `maxMintFeeStart` (against what the actual calculations dictate in the `_computeVariableOpeningFee()`): 

```solidity
// BigBang.sol

function setMinAndMaxMintRange(uint256 _min, uint256 _max) external onlyOwner {
        emit UpdateMinMaxMintRange(minMintFeeStart, _min, maxMintFeeStart, _max);

        if (_min >= _max) revert NotValid(); 

        minMintFeeStart = _min;
        maxMintFeeStart = _max;
    } 
```

This will make it impossible to properly update the `maxMintFeeStart` and `minMintFeeStart` to have proper values because if it is enforced that `maxMintFeeStart` > than `minMintFeeStart`, then `_computeVariableOpeningFee()` will always enter the first `if (_exchangeRate >= minMintFeeStart)` and wrongly return the minimum fee.

## Impact

Medium. Although this looks like a bug that doesn’t have a big impact in the protocol, it actually does. The fees will always be wrongly applied for collaterals different from stablecoins, and applying these kind of fees when borrowing is one of the core mechanisms to keep USDO peg, as described in [Tapioca’s documentation](https://docs.tapioca.xyz/tapioca/core-technologies/big-bang#variable-usdo-creation-fee). If this mechanisms doesn’t work properly, users won’t be properly incentivized to borrow/repay considering the different market conditions that might take place and affect USDO’s peg to $1.

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLendingCommon.sol#L87-L91

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BigBang.sol#L194-L195

## Tool used

Manual Review

## Recommendation

The mitigation for this is straightforward. Change the `setMinAndMaxMintRange()` function so that `_max` is enforced to be smaller than `_min`:

```diff
// BigBang.sol

function setMinAndMaxMintRange(uint256 _min, uint256 _max) external onlyOwner {
        emit UpdateMinMaxMintRange(minMintFeeStart, _min, maxMintFeeStart, _max);

-        if (_min >= _max) revert NotValid(); 
+        if (_max >= _min) revert NotValid(); 

        minMintFeeStart = _min;
        maxMintFeeStart = _max;
    } 
```

Also, I would recommend not to hardcode the values of `maxMintFeeStart` and `minMintFeeStart` and pass them as parameter instead, inside `_initCoreStorage()` , as they should always be different considering the collateral configured for that market.
