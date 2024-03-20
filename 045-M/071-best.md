Rural Amethyst Tapir

medium

# Stargate Pools conversion rate leads to token accumulation inside the Balancer contract

## Summary
Stargate pools conversion rate leads to token accumulation inside the `Balancer` contract and dangling allowances to the StargateRouter contract. This breaks the expected behavior of the rebalancing process and can result in a loss of tokens.

## Vulnerability Detail
Stargate pools have a concept of convert rate. It's calculated based on the `sharedDecimals` and `localDecimals` for a specific pool. For example, the DAI Pool has the `sharedDecimals` set to 6 while `localDecimals` is 18.

The convert rate is then: `10^(localDecimals - sharedDecimals) = 10^12`.

Here is the [DAI Pool](https://etherscan.io/address/0x0Faf1d2d3CED330824de3B8200fc8dc6E397850d#readContract) on Ethereum and the convert rate logic inside the [Pool contract](https://github.com/stargate-protocol/stargate/blob/5f0dfd2/contracts/Pool.sol#L140).

During the rebalancing process:
- the specified amount is extracted from the `mTOFT`
- allowance is set for that amount to the StargateRouter contract
- the rebalance amount is deducted 
- Stargate transfer is invoked.

However, if the specified amount is not a multiple of the conversion rate, which in the case of DAI pool is `10^12`, the consequence is:

- There will be an unspent allowance from Balancer to the StargateRouter contract.
- The remaining amount of tokens will accumulate inside the Balancer contract.

Repeatedly calling the `rebalance` function will leave more and more tokens inside the `Balancer` contract while leaving dangling allowances to the StargateRouter contract.

In case there is an issue upstream inside the StargateRouter contract it could result in a loss of tokens accumulated inside the Balancer contract. 

## Impact
ERC20 tokens will accumulate inside the Balancer contract with dangling allowances left to the StargateRouter contract. Under certain conditions, this can result in a loss of tokens.

## Code Snippet

## Tool used

Manual Review

## Recommendation
The recommendation is to add a check for the conversion rate and adjust the amount to be rebalanced accordingly.

```diff
+
+
+interface IStargatePool {
+    function convertRate() external view returns (uint256);
+}
+
+interface IStargateFactory {
+    function getPool(uint256 _poolId) external view returns (address);
+}
+
 contract Balancer is Ownable {
     using SafeERC20 for IERC20;
 
     IStargateRouter public immutable routerETH;
     IStargateRouter public immutable router;
+    IStargateFactory public immutable stargateFactory;
 
-    constructor(address _routerETH, address _router, address _owner) {
+    constructor(address _routerETH, address _router, address sgFactory, address _owner) {
         if (_router == address(0)) revert RouterNotValid();
         if (_routerETH == address(0)) revert RouterNotValid();
         routerETH = IStargateRouter(_routerETH);
         router = IStargateRouter(_router);
+        stargateFactory = IStargateFactory(sgFactory);
 
         transferOwnership(_owner);
         rebalancer = _owner;
@@ -179,8 +191,14 @@ contract Balancer is Ownable {
             revert RebalanceAmountNotSet();
         }
 
+        uint256 convertedAmount = _amount;
+        uint256 srcPoolId = connectedOFTs[_srcOft][_dstChainId].srcPoolId;
+        address stargatePool = stargateFactory.getPool(srcPoolId);
+        uint256 convertRate = IStargatePool(stargatePool).convertRate();
+        if (convertRate != 1) { convertedAmount = (_amount / convertRate) * convertRate; }
+
         //extract
-        ITOFT(_srcOft).extractUnderlying(_amount);
+        ITOFT(_srcOft).extractUnderlying(convertedAmount);
 
             if (msg.value == 0) revert FeeAmountNotSet();
             if (_isNative) {
                 if (disableEth) revert SwapNotEnabled();
-                _sendNative(_srcOft, _amount, _dstChainId, _slippage);
+                _sendNative(_srcOft, convertedAmount, _dstChainId, _slippage);
             } else {
-                _sendToken(_srcOft, _amount, _dstChainId, _slippage, _ercData);
+                _sendToken(_srcOft, convertedAmount, _dstChainId, _slippage, _ercData);
             }
 
-            connectedOFTs[_srcOft][_dstChainId].rebalanceable -= _amount;
-            emit Rebalanced(_srcOft, _dstChainId, _slippage, _amount, _isNative);
+            connectedOFTs[_srcOft][_dstChainId].rebalanceable -= convertedAmount;
+            emit Rebalanced(_srcOft, _dstChainId, _slippage, convertedAmount, _isNative);
         }
     }

```
