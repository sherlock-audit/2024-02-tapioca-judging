Rural Amethyst Tapir

high

# `mTOFT` can be forced to receive the wrong ERC20 leading to token lockup

## Summary
Due to Stargate's functionality of swapping one token on the source chain to another token on the destination chain, it is possible to force `mTOFT` to receive the wrong ERC20 token leading to token lockup.

## Vulnerability Detail
Stargate allows for swapping between different tokens. These are usually correlated stablecoins. They are defined as **Stargate Chains Paths** inside the docs: https://stargateprotocol.gitbook.io/stargate/developers/stargate-chain-paths.

To give an example, a user can:

- Provide USDC on Ethereum and receive USDT on Avalanche. 
- Provide USDC on Avalanche and receive USDT on Arbitrum. 
- etc.

This can also be observed by just playing around with the Stargate UI: https://stargate.finance/transfer.

The `Balancer.sol` contract initializes the connected OFTs through the [`initConnectedOFT`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/Balancer.sol#L224) function.
This function is only callable by the admin and he specifies the src and dst pool ids. PoolIds refer to a specific StargatePool that holds the underlying asset(USDC, USDT, etc.): https://stargateprotocol.gitbook.io/stargate/developers/pool-ids.

The issue here is that poolIds are not enforced during the rebalancing process. As it can be observed the `bytes memory _ercData` is not checked for its content.

```solidity
## Balancer.sol

function _sendToken(
        address payable _oft,
        uint256 _amount,
        uint16 _dstChainId,
        uint256 _slippage,
>>>        bytes memory _data
    ) private {
        address erc20 = ITOFT(_oft).erc20();
        if (IERC20Metadata(erc20).balanceOf(address(this)) < _amount) {
            revert ExceedsBalance();
        }
        {
>>>            (uint256 _srcPoolId, uint256 _dstPoolId) = abi.decode(_data, (uint256, uint256));
            _routerSwap(_dstChainId, _srcPoolId, _dstPoolId, _amount, _slippage, _oft, erc20);
        }
    }

```

It is simply decoded and passed as is. 

This is a problem and imagine the following scenario:

1. A Gelato bot calls the rebalance method for `mTOFT` that has USDC as erc20 on Ethereum.
2. The bot encodes the `ercData` so `srcChainId = 1` pointing to USDC but `dstChainId = 2` pointing to USDT on Avalanche.
3. Destination `mTOFT` is fetched from [`connectedOFTs`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/Balancer.sol#L307) and points to the mTOFT with USDC as erc20 on Avalanche.
4. Stargate will take USDC on Ethereum and provide USDT on Avalanche.
5. mTOFT with USDC as underlying erc20 on Avalanche will receive USDT token and it will remain lost as the balance of the `mTOFT` contract. 

As this is a clear path for locking up wrong tokens inside the `mTOFT` contract, it is a critical issue.

## Impact
The impact of this vulnerability is critical. It allows for locking up wrong tokens inside the mTOFT contract causing irreversible loss of funds. 

## Code Snippet
- https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/Balancer.sol#L293

## Tool used

Manual Review

## Recommendation
The `initConnectedOFT` function should enforce the poolIds for the src and dst chains.The rebalance function should just fetch these saved values and use them.

```diff
 
@@ -164,14 +176,12 @@ contract Balancer is Ownable {
      * @param _dstChainId the destination LayerZero id
      * @param _slippage the destination LayerZero id
      * @param _amount the rebalanced amount
-     * @param _ercData custom send data
      */
     function rebalance(
         address payable _srcOft,
         uint16 _dstChainId,
         uint256 _slippage,
-        uint256 _amount,
-        bytes memory _ercData
+        uint256 _amount
     ) external payable onlyValidDestination(_srcOft, _dstChainId) onlyValidSlippage(_slippage) {
         {
@@ -188,13 +204,13 @@ contract Balancer is Ownable {
             if (msg.value == 0) revert FeeAmountNotSet();
             if (_isNative) {
                 if (disableEth) revert SwapNotEnabled();
                 _sendNative(_srcOft, _amount, _dstChainId, _slippage);
             } else {
-                _sendToken(_srcOft, _amount, _dstChainId, _slippage, _ercData);
+                _sendToken(_srcOft, _amount, _dstChainId, _slippage);
             }

 
@@ -221,7 +237,7 @@ contract Balancer is Ownable {
      * @param _dstOft the destination TOFT address
      * @param _ercData custom send data
      */
-    function initConnectedOFT(address _srcOft, uint16 _dstChainId, address _dstOft, bytes memory _ercData)
+    function initConnectedOFT(address _srcOft, uint256 poolId, uint16 _dstChainId, address _dstOft, bytes memory _ercData)
         external
         onlyOwner
     {
@@ -231,10 +247,8 @@ contract Balancer is Ownable {
         bool isNative = ITOFT(_srcOft).erc20() == address(0);
         if (!isNative && _ercData.length == 0) revert PoolInfoRequired();
 
-        (uint256 _srcPoolId, uint256 _dstPoolId) = abi.decode(_ercData, (uint256, uint256));
-
         OFTData memory oftData =
-            OFTData({srcPoolId: _srcPoolId, dstPoolId: _dstPoolId, dstOft: _dstOft, rebalanceable: 0});
+            OFTData({srcPoolId: poolId, dstPoolId: poolId, dstOft: _dstOft, rebalanceable: 0});
 
         connectedOFTs[_srcOft][_dstChainId] = oftData;
         emit ConnectedChainUpdated(_srcOft, _dstChainId, _dstOft);
 
     function _sendToken(
         address payable _oft,
         uint256 _amount,
         uint16 _dstChainId,
-        uint256 _slippage,
-        bytes memory _data
+        uint256 _slippage
     ) private {
         address erc20 = ITOFT(_oft).erc20();
         if (IERC20Metadata(erc20).balanceOf(address(this)) < _amount) {
             revert ExceedsBalance();
-        }
+            }
         {
-            (uint256 _srcPoolId, uint256 _dstPoolId) = abi.decode(_data, (uint256, uint256));
-            _routerSwap(_dstChainId, _srcPoolId, _dstPoolId, _amount, _slippage, _oft, erc20);
+            _routerSwap(_dstChainId, _amount, _slippage, _oft, erc20);
         }
     }
 
     function _routerSwap(
         uint16 _dstChainId,
-        uint256 _srcPoolId,
-        uint256 _dstPoolId,
         uint256 _amount,
         uint256 _slippage,
         address payable _oft,
         address _erc20
     ) private {
         bytes memory _dst = abi.encodePacked(connectedOFTs[_oft][_dstChainId].dstOft);
+        uint256 poolId = connectedOFTs[_oft][_dstChainId].srcPoolId;
         IERC20(_erc20).safeApprove(address(router), _amount);
         router.swap{value: msg.value}(
             _dstChainId,
-            _srcPoolId,
-            _dstPoolId,
+            poolId,
+            poolId,
             payable(this),
             _amount,
             _computeMinAmount(_amount, _slippage),
```

Admin is trusted but you can optionally add additional checks inside the `initConnectedOFT` function to ensure that the poolIds are correct for the src and dst mTOFTs. 
