Funny Ginger Squid

medium

# Share computing for reward distribution is incorrect

## Summary

Share computing for reward distribution is incorrect

## Vulnerability Detail

[Line of code](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/Penrose.sol#L565)
```solidity
function _depositFeesToTwTap(IMarket market, ITwTap twTap) private {
        if (!isMarketRegistered[address(market)]) revert NotValid();

        uint256 feeShares = market.refreshPenroseFees();
        if (feeShares == 0) return;

        address _asset = market.asset();
        uint256 _assetId = market.assetId();
        
        yieldBox.withdraw(_assetId, address(this), address(this), 0, feeShares);

        uint256 rewardTokenId = twTap.rewardTokenIndex(_asset);
        uint256 feeAmount = yieldBox.toAmount(_assetId, feeShares, false);
        _distributeOnTwTap(feeAmount, rewardTokenId, _asset, twTap);
    }
```
This code in `Penrose.sol` is used to distribute the reward to `twTAP.sol` user that lock their TAP token

but the problem is that we are calling
```solidity
   yieldBox.withdraw(_assetId, address(this), address(this), 0, feeShares);
```
and ignored the return parameter: 

[Line of code](https://github.com/Tapioca-DAO/tap-yieldbox/blob/8de8ad15fe80c4e00843475d7d20fae4d0397003/contracts/interfaces/IYieldBox.sol#L56)
```solidity
function withdraw(
        uint256 assetId,
        address from,
        address to,
        uint256 amount,
        uint256 share
    ) external returns (uint256 amountOut, uint256 shareOut);
```
the returned `amountOut` should be what used for reward distribution

but the code compute the `feeAmount` again using stale shares
```solidity
     uint256 feeAmount = yieldBox.toAmount(_assetId, feeShares, false);
     _distributeOnTwTap(feeAmount, rewardTokenId, _asset, twTap);
```
if `feeAmount` is greater than `amountOut`, calling `_distributeOnTwTa`p will revert and block reward distribution.

[Line of code](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/Penrose.sol#L580)
```solidity
function _distributeOnTwTap(uint256 amount, uint256 rewardTokenId, address _asset, ITwTap twTap) private {
        _asset.safeApprove(address(twTap), amount);
        twTap.distributeReward(rewardTokenId, amount);
        emit LogTwTapFeesDeposit(amount);
    }
```

## Impact

.

## Code Snippet

1. https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/Penrose.sol#L565

3. https://github.com/Tapioca-DAO/tap-yieldbox/blob/8de8ad15fe80c4e00843475d7d20fae4d0397003/contracts/interfaces/IYieldBox.sol#L56

5. https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/Penrose.sol#L580

## Tool used

Manual Review

## Recommendation

```solidity
(uint256 amountOut, )yieldBox.withdraw(_assetId, address(this), address(this), 0, feeShares);

uint256 rewardTokenId = twTap.rewardTokenIndex(_asset);

_distributeOnTwTap(amountOut, rewardTokenId, _asset, twTap);
```
