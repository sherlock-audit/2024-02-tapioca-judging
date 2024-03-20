Modern Mandarin Wasp

medium

# Singularity::setSingularityConfig Wrong liquidationCollateralizationRate is set

## Summary

the function setSingularityConfig enables to set some configuration variables on the Singularity contract. However the wrong liquidationCollateralizationRate is set, which could lead to unwanted liquidations.

## Vulnerability Detail
We can see here that the variable which is set in `setSingularityConfig` [lqCollateralizationRate](https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/Singularity.sol#L358) is not used anywhere.
 
## Impact
The wrong variable is set for liquidation collateralization rate, and can cause a wrong value for the actually used variable (`liquidationCollateralizationRate`) if set this way.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Set the actual variable `liquidationCollateralizationRate`, and remove this unused variable