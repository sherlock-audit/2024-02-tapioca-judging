Round Pearl Crow

medium

# WETH was never set in baseLeverageExecutor.sol

## Summary
WETH State Var was never set in baseLeverageExecutor.sol
## Vulnerability Details
see summary
## Impact
WETH will be address zero, it won't be possible to wrap and unwrap ETH
## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/leverage/BaseLeverageExecutor.sol#L47
## Tool used

Manual Review

## Recommendation
initialize the WETH state Var via the constructor.