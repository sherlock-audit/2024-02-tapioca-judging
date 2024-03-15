Modern Mandarin Wasp

high

# BBLiquidation::_liquidateUser liquidator can bypass protocol fee on liquidation by returning returnedShare == borrowShare

## Summary
When a liquidator liquidates a position on BigBang/Singularity market, they do not get the full liquidationBonus amount, but a callerShare which depends on the efficiency of the liquidation. 

However since this share is taken after the liquidator has swapped the seized collateral to an asset amount, the liquidator can simply choose to return enough asset to repay the borrow, reducing the extra amount to zero.

In that case the protocol fee and the caller share would be zero, but the liquidator has seized the full liquidation bonus during the swap.

## Vulnerability Detail

We can see that after the collateral has been seized from the liquidatee, the full amount of collateral with the liquidation bonus is sent to an arbitrary `liquidationReceiver` during `_swapCollateralWithAsset`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLiquidation.sol#L262-L263

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLiquidation.sol#L152-L159


The liquidator can choose to send back only `borrowAmount` of asset, effectively keeping excess collateral to himself

## Impact
Liquidator steals the due fee from protocol during liquidation 

## Code Snippet

## Tool used

Manual Review

## Recommendation
Consider adding a slippage control to the swap executed by the liquidator (e.g the liquidator must return at least 105% of `borrowAmount`, when seizing 110% of equivalent collateral)