Winning Cobalt Barracuda

high

# The borrowing approval of the market is risky for users, as the spender can steal unlimited funds of user with a small allowance

## Summary
The `buyCollateral` function allows the sender to borrow on behalf of the user and pull asset tokens as supply asset from the user to buy collateral. The requirement is simply that the sender needs to have a borrow allowance from the user greater than the received collateral value. However, a malicious sender with only a small borrow allowance can attempt to pull an unlimited amount of supply asset from the user while manipulating the swap to obtain a small value of collateral. Therefore, a malicious sender can steal unlimited funds from the user with only 1 wei of borrow allowance.
## Vulnerability Detail
In `SGLLeverage.buyCollateral()` function: 
```solidity=
uint256 supplyShare = yieldBox.toShare(assetId, calldata_.supplyAmount, true);
uint256 supplyShareToAmount;
if (supplyShare > 0) {
    (supplyShareToAmount,) =
        yieldBox.withdraw(assetId, calldata_.from, address(leverageExecutor), 0, supplyShare);
}
(, uint256 borrowShare) = _borrow(calldata_.from, address(this), calldata_.borrowAmount);

(uint256 borrowShareToAmount,) =
    yieldBox.withdraw(assetId, address(this), address(leverageExecutor), 0, borrowShare);
amountOut = leverageExecutor.getCollateral(
    collateralId,
    address(asset),
    address(collateral),
    supplyShareToAmount + borrowShareToAmount,
    calldata_.from,
    calldata_.data
);
uint256 collateralShare = yieldBox.toShare(collateralId, amountOut, false);
if (collateralShare == 0) revert CollateralShareNotValid();
address(asset).safeApprove(address(yieldBox), type(uint256).max);
yieldBox.depositAsset(collateralId, address(this), address(this), 0, collateralShare); // TODO Check for rounding attack?
address(asset).safeApprove(address(yieldBox), 0);

_allowedBorrow(calldata_.from, collateralShare);
```
This function attempts to pull supply asset directly from the user (`callData._from`) and borrow on behalf of the user to obtain assets. Afterward, it tries to swap the total of the supply asset and borrowed asset, which were withdrawn to the leverageExecutor contract, to obtain collateral tokens. The borrowing allowance amount that the sender needs to be approved is the received collateral share.

The problem is that the sender can pull a lot of supply assets from the user while manipulating `leverageExecutor.getCollateral()` with malicious data to receive a small collateral share. The leverageExecutor contract will approve and swap based on the passed data, so the malicious sender can steal unlimited supply assets while returning very few collaterals back to Singularity. This results in the needed allowance of the sender being manipulated to be very small (1 wei in the example, just to ensure collateralShare != 0).

The same vulnerability exists in BBLeverage contract.
## Impact
The borrowing approval of the market is risky for users, as the spender can steal unlimited funds from the user with only 1 wei of allowance.

## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLeverage.sol#L67-L86
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLeverage.sol#L91

## Tool used

Manual Review

## Recommendation
Should convert the used assets (supplyShareToAmount + borrowShareToAmount) to calculate the collateral value required for the spender's approval.