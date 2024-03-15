Zealous Pineapple Duck

medium

# Unpausing with accrue timestamp reset can remove the accrual between last recorded accrue time and pausing time

## Summary

Updating pause to `false` have an option of resetting the accrual timestamp. It can backfire if there was a substantial enough period of not updating the accrual before pausing, as it does not call accrue by itself.

## Vulnerability Detail

In other words `updatePause(type, false, true)` will erase interest accrual for the unpaused period between last accrual and pausing. That period can be arbitrary long.

## Impact

Interest accrual is incorrectly erased for the period before pause was initiated. This is protocol-wide break of core logic via loss of yield for that period, so the impact is high. The preconditions are that `resetAccrueTimestmap == true` must be used on unpausing and that long enough period without accrual call should take place before pausing. The probability on that can be estimated as low.

Likelihood: Low + Impact: High = Severity: Medium.

## Code Snippet

If `resetAccrueTimestmap == true` on unpausing the accrual between `accrueInfo.lastAccrued` and pausing time is lost:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/Singularity.sol#L261-L272

```solidity
    function updatePause(PauseType _type, bool val, bool resetAccrueTimestmap) external {
        if (msg.sender != conservator) revert NotAuthorized();
        if (val == pauseOptions[_type]) revert SameState();
        emit PausedUpdated(_type, pauseOptions[_type], val);
        pauseOptions[_type] = val;

        // In case of 'unpause', `lastAccrued` is set to block.timestamp
        // Valid for all action types that has an impact on debt or supply
        if (!val && (_type != PauseType.AddCollateral && _type != PauseType.RemoveCollateral)) {
>>          accrueInfo.lastAccrued = resetAccrueTimestmap ? block.timestamp.toUint64() : accrueInfo.lastAccrued;
        }
    }
```

That's incorrect as that was a going concern period for which lenders should have interest accounted for.

## Tool used

Manual Review

## Recommendation

Consider accruing whenever pause is being triggered, so the state be updated as of pausing time:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/Singularity.sol#L261-L272

```diff
    function updatePause(PauseType _type, bool val, bool resetAccrueTimestmap) external {
        if (msg.sender != conservator) revert NotAuthorized();
        if (val == pauseOptions[_type]) revert SameState();
        emit PausedUpdated(_type, pauseOptions[_type], val);
        pauseOptions[_type] = val;
+       // since the `lastAccrued` can be reset later the state need to be updated as of pausing time
+       if (val) {
+           _accrue();
+       }
        // In case of 'unpause', `lastAccrued` is set to block.timestamp
        // Valid for all action types that has an impact on debt or supply
        if (!val && (_type != PauseType.AddCollateral && _type != PauseType.RemoveCollateral)) {
            accrueInfo.lastAccrued = resetAccrueTimestmap ? block.timestamp.toUint64() : accrueInfo.lastAccrued;
        }
    }
```