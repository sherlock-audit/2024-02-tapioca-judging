Rural Amethyst Tapir

high

# `TOFTMarketReceiverModule::marketBorrowReceiver` flow is broken

## Summary

The `TOFTMarketReceiverModule::marketBorrowReceiver` flow is broken and will revert when the Magnetar contract tries to transfer the ERC1155 tokens to the Market contract.

## Vulnerability Detail

[`TOFTMarketReceiverModule::marketBorrowReceiver`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/modules/TOFTMarketReceiverModule.sol#L108) flow is broken.

Let's examine it more closely:

- After checking the whitelisting status for the `marketHelper`, `magnetar` and the `market` contracts an approval is made to the Magnetar contract. 
- [`MagnetarCollateralModule::depositAddCollateralAndBorrowFromMarket`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/modules/TOFTMarketReceiverModule.sol#L142) get called with the passed parameters.
- If the `data.deposit` is true, the Magnetar contract will call `_extractTokens` with the following params: `from = msg_.user`, `token = collateralAddress` and `amount = msg_.collateralAmount`.

```solidity
    function _extractTokens(address _from, address _token, uint256 _amount) internal returns (uint256) {
        uint256 balanceBefore = IERC20(_token).balanceOf(address(this));
        // IERC20(_token).safeTransferFrom(_from, address(this), _amount);
        pearlmit.transferFromERC20(_from, address(this), address(_token), _amount);
        uint256 balanceAfter = IERC20(_token).balanceOf(address(this));
        if (balanceAfter <= balanceBefore) revert Magnetar_ExtractTokenFail();
        return balanceAfter - balanceBefore;
    }
```
- The collateral gets transferred into the Magnetar contract in case the `msg._user` has given sufficient allowance to the Magnetar contract through the Pearlmit contract.
- After this `_setApprovalForYieldBox(data.market, yieldBox_);` is called that sets the allowance of the Magnetar contract to the Market contract. 
- Then `addCollateral` is called on the Market contract. I've inlined the internal function to make it easier to follow:
```solidity
    function _addCollateral(address from, address to, bool skim, uint256 amount, uint256 share) internal {
        if (share == 0) {
            share = yieldBox.toShare(collateralId, amount, false);
        }
        uint256 oldTotalCollateralShare = totalCollateralShare;
        userCollateralShare[to] += share;
        totalCollateralShare = oldTotalCollateralShare + share;

        // yieldBox.transfer(from, address(this), _assetId, share);
        bool isErr = pearlmit.transferFromERC1155(from, address(this), address(yieldBox), collateralId, share);
        if (isErr) {
            revert TransferFailed();
        }
    }
```

- After the `userCollateralShare` mapping is updated [`pearlmit.transferFromERC1155(from, address(this), address(yieldBox), collateralId, share);`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/markets/singularity/SGLCommon.sol#L172) gets called. 
- This is critical as now the Magnetar is supposed to transfer the ERC1155 tokens(Yieldbox) to the Market contract.
- In order to do this the Magnetar contract should have given the allowance to the Market contract through the Pearlmit contract.
- This is not the case, the Magnetar has only executed `_setApprovalForYieldBox(data.market, yieldBox_);`, nothing else.
- It will revert inside the Pearlmit contract `transferFromERC1155` function when the allowance is being checked.

### Other occurrences

1. `TOFT::mintLendXChainSGLXChainLockAndParticipateReceiver` has a similar issue as:
- Extract the bbCollateral from the user, sets approval for the BigBang contract through YieldBox.
- But then inside the `BBCollateral::addCollateral` the [`_addTokens`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/markets/bigBang/BBCommon.sol#L133) again expects an allowance through the Pearlmit contract.  


2. `TOFT::lockAndParticipateReceiver` calls the `Magnetar:lockAndParticipate` where:

```solidity

## MagnetarMintCommonModule.sol

function _lockOnTOB(
        IOptionsLockData memory lockData,
        IYieldBox yieldBox_,
        uint256 fraction,
        bool participate,
        address user,
        address singularityAddress
    ) internal returns (uint256 tOLPTokenId) {
    ....
        _setApprovalForYieldBox(lockData.target, yieldBox_);
        tOLPTokenId = ITapiocaOptionLiquidityProvision(lockData.target).lock(
             participate ? address(this) : user, singularityAddress, lockData.lockDuration, lockData.amount
        );
}

## TapiocaOptionLiquidityProvision.sol

function lock(address _to, IERC20 _singularity, uint128 _lockDuration, uint128 _ybShares)
    external
    nonReentrant
    returns (uint256 tokenId)
{
    // Transfer the Singularity position to this contract
    // yieldBox.transfer(msg.sender, address(this), sglAssetID, _ybShares);
    {
        bool isErr =
            pearlmit.transferFromERC1155(msg.sender, address(this), address(yieldBox), sglAssetID, _ybShares);
        if (isErr) {
            revert TransferFailed();
        }
    }
```
- The same issue where approval through the Pearlmit contract is expected.


## Impact

The `TOFTMarketReceiverModule::marketBorrowReceiver` flow is broken and will revert when the Magnetar contract tries to transfer the ERC1155 tokens to the Market contract. There are also other instances of similar issues. 

## Code Snippet

- https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/markets/bigBang/BBCommon.sol#L133
- https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/modules/TOFTMarketReceiverModule.sol#L108

## Tool used

Manual Review

## Recommendation

Review all the allowance mechanisms and ensure that they are correct. 