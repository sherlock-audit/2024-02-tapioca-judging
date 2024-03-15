Modern Mandarin Wasp

high

# BBLiquidation/SGLLiquidation::_updateBorrowAndCollateralShare liquidator can bypass bad debt handling to ensure whole liquidation reward

## Summary
When handling liquidation in BigBang and Singularity market, the protocol ensures that the liquidatee has enough collateral to cover for the liquidation and reward. If that's not the case, then the reward for the liquidator is shrunk proportionally to the bad debt incurred by the protocol.

However the liquidator can simply choose to bypass this protection by setting a max repay amount small enough so it can be covered by the collateral of the liquidatee. This enables the liquidator to get full reward on a partial liquidation, and leaves the protocol with only bad debt

## Vulnerability Detail
We can see that some logic for handling bad debt is implemented [here](https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLiquidation.sol#L201-L218) when `collateralPartInAsset < borrowAmountWithBonus`

However the liquidator can reduce the amount to repay arbitrarily by setting the [maxBorrowPart](https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLiquidation.sol#L188) parameter.

Thus the liquidator can always choose to execute the [second branch](https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLiquidation.sol#L219-L223), ensuring full reward.

## Impact
The protocol incurs more bad debt than due because liquidator can bypass bad debt protection mechanism

## Code Snippet

## Tool used

Manual Review

## Recommendation
Ensure a minimal repay amount in order for the liquidation to always make the account solvent, this would make it impossible for the liquidator to reduce the repay amount arbitrarily