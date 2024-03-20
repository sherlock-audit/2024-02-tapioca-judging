Winning Cobalt Barracuda

high

# `_computeClosingFactor` function will return incorrect values, lower than needed, because it uses `collateralizationRate` to calculate the denominator

## Summary
`_computeClosingFactor` is used to calculate the required borrow amount that should be liquidated to make the user's position solvent. However, this function uses `collateralizationRate` (defaulting to 75%) to calculate the liquidated amount, while the threshold to be liquidatable is `liquidationCollateralizationRate` (defaulting to 80%). Therefore, it will return incorrect liquidated amount.
## Vulnerability Detail
In `_computeClosingFactor` of Market contract:
```solidity=
//borrowPart and collateralPartInAsset should already be scaled due to the exchange rate computation
uint256 liquidationStartsAt =
    (collateralPartInAsset * _liquidationCollateralizationRate) / (10 ** ratesPrecision);///80% collateral value in asset in default

if (borrowPart < liquidationStartsAt) return 0;

//compute numerator
uint256 numerator = borrowPart - liquidationStartsAt;
//compute denominator
uint256 diff =
    (collateralizationRate * ((10 ** ratesPrecision) + _liquidationMultiplier)) / (10 ** ratesPrecision);
int256 denominator = (int256(10 ** ratesPrecision) - int256(diff)) * int256(1e13);

//compute closing factor
int256 x = (int256(numerator) * int256(1e18)) / denominator;
```
A user will be able to be liquidated if their ratio between borrow and collateral value exceeds `liquidationCollateralizationRate` (see [`_isSolvent()`](https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/Market.sol#L476-L477) function).
However, `_computeClosingFactor` uses `collateralizationRate`  (defaulting to 75%) to calculate the denominator for the needed liquidate amount, while the numerator is calculated by using `liquidationCollateralizationRate` (80% in default). These variables were initialized in [`_initCoreStorage()`](https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BigBang.sol#L176-L178).

In the above calculation of `_computeClosingFactor` function, in default:
`_liquidationMultiplier` = 12%,
`numerator` = `borrowPart` - `liquidationStartsAt` = borrowAmount - 80% * collateralToAssetAmount 
=> x will be: **numerator / (1 - 75% * 112%) = numerator / 16%**

However, during a partial liquidation of BigBang or Singularity, the actual collateral bonus is `liquidationBonusAmount`, defaulting to 10%. ([code snippet](https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLiquidation.sol#L197-L199)). Therefore, the minimum liquidated amount required to make user solvent (unable to be liquidated again) is: **numerator / (1 - 80% * 110%) = numerator / 12%**.

As result, `computeClosingFactor()` function will return a lower liquidated amount than needed to make user solvent, even when that function attempts to over-liquidate with  `_liquidationMultiplier` > `liquidationBonusAmount`.

## Impact
This issue will result in the user still being liquidatable after a partial liquidation because it liquidates a lower amount than needed. Therefore, the user will never be solvent again after they are undercollateralized until their position is fully liquidated. This may lead to the user being liquidated more than expected, or experiencing a loss of funds in attempting to recover their position.
## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/Market.sol#L325-L326

## Tool used

Manual Review

## Recommendation
Use `liquidationCollateralizationRate` instead of `collateralizationRate` to calculate the denominator in `_computeClosingFactor`