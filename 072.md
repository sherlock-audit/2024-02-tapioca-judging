Rural Amethyst Tapir

medium

# Gas parameters for Stargate swap are hardcoded leading to stuck messages

## Summary
The `dstGasForCall` for transferring erc20s through Stargate is hardcoded to 0 in the `Balancer` contract leading to `sgReceive` not being called during Stargate swap.
As a consequence, the `sgReceive` has to be manually called to clear the `cachedSwapLookup` mapping, but this can be DoSed due to the fact that the `mTOFT::sgReceive` doesn't validate any of its parameters.
This can be exploited to perform a long-term DoS attack. 

## Vulnerability Detail
### Gas parameters for Stargate

Stargate Swap allows the caller to specify the:

- `dstGasForCall` which is the gas amount forwarded while calling the `sgReceive` on the destination contract.
- `dstNativeAmount` and `dstNativeAddr` which is the amount and address where the native token is sent to.

Inside the `Balancer.sol` contract, the [`dstGasForCall` is hardcoded to 0](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/Balancer.sol#L316). 
The `dstGasForCall` gets forwarded from Stargate `Router` into the Stargate [`Bridge`](https://github.com/stargate-protocol/stargate/blob/c647a3a647fc693c38b16ef023c54e518b46e206/contracts/Bridge.sol#L115) contract.

```solidity
    function swap(
        uint16 _chainId,
        uint256 _srcPoolId,
        uint256 _dstPoolId,
        address payable _refundAddress,
        Pool.CreditObj memory _c,
        Pool.SwapObj memory _s,
>>>>>>        IStargateRouter.lzTxObj memory _lzTxParams, 
        bytes calldata _to,
        bytes calldata _payload
    ) external payable onlyRouter {
>>>>>>        bytes memory payload = abi.encode(TYPE_SWAP_REMOTE, _srcPoolId, _dstPoolId, _lzTxParams.dstGasForCall, _c, _s, _to, _payload);
        _call(_chainId, TYPE_SWAP_REMOTE, _refundAddress, _lzTxParams, payload);
    }

    function _call(
        uint16 _chainId,
        uint8 _type,
        address payable _refundAddress,
        IStargateRouter.lzTxObj memory _lzTxParams,
        bytes memory _payload
    ) internal {
        bytes memory lzTxParamBuilt = _txParamBuilder(_chainId, _type, _lzTxParams);
        uint64 nextNonce = layerZeroEndpoint.getOutboundNonce(_chainId, address(this)) + 1;
        layerZeroEndpoint.send{value: msg.value}(_chainId, bridgeLookup[_chainId], _payload, _refundAddress, address(this), lzTxParamBuilt);
        emit SendMsg(_type, nextNonce);
    }
```

It gets encoded inside the payload that is sent through the LayerZero message. 
The payload gets decoded inside the [`Bridge::lzReceive`](https://github.com/stargate-protocol/stargate/blob/c647a3a647fc693c38b16ef023c54e518b46e206/contracts/Bridge.sol#L79) on destination chain. 
And `dstGasForCall` is forwarded to the [`sgReceive`](https://github.com/stargate-protocol/stargate/blob/c647a3a647fc693c38b16ef023c54e518b46e206/contracts/Router.sol#L406) function:

```solidity
## Bridge.sol

 function lzReceive(
        uint16 _srcChainId,
        bytes memory _srcAddress,
        uint64 _nonce,
        bytes memory _payload
    ) external override {
        if (functionType == TYPE_SWAP_REMOTE) {
            (
                ,
                uint256 srcPoolId,
                uint256 dstPoolId,
>>>>>                uint256 dstGasForCall,
                Pool.CreditObj memory c,
                Pool.SwapObj memory s,
                bytes memory to,
                bytes memory payload
            ) = abi.decode(_payload, (uint8, uint256, uint256, uint256, Pool.CreditObj, Pool.SwapObj, bytes, bytes));
```

If it is zero like in the `Balancer.sol` contract or its value is too small the `sgReceive` will fail, but the payload will be saved in the `cachedSwapLookup` mapping. At the same time the tokens are transferred to the destination contract, which is the `mTOFT`.
Now anyone can call the `sgReceive` manually through the [`clearCachedSwap`](https://github.com/stargate-protocol/stargate/blob/c647a3a647fc693c38b16ef023c54e518b46e206/contracts/Router.sol#L285) function:

```solidity
    function clearCachedSwap(
        uint16 _srcChainId,
        bytes calldata _srcAddress,
        uint256 _nonce
    ) external {
        CachedSwap memory cs = cachedSwapLookup[_srcChainId][_srcAddress][_nonce];
        require(cs.to != address(0x0), "Stargate: cache already cleared");
        // clear the data
        cachedSwapLookup[_srcChainId][_srcAddress][_nonce] = CachedSwap(address(0x0), 0, address(0x0), "");
        IStargateReceiver(cs.to).sgReceive(_srcChainId, _srcAddress, _nonce, cs.token, cs.amountLD, cs.payload);
    }
```

Although not the intended behavior there seems to be no issue with erc20 token sitting on the `mTOFT` contract for a shorter period of time.

### sgReceive

This leads to the second issue. The `sgReceive` function interface specifies the `chainId`, `srcAddress`, and `token`.
- `chainId` is the layerZero chainId of the source chain. In their docs referred to endpointId: https://layerzero.gitbook.io/docs/technical-reference/mainnet/supported-chain-ids
- `srcAddress` is the address of the source sending contract
- `token` is the address of the token that was sent to the destination contract. 

In the current implementation, the `sgReceive` function doesn't check any of these parameters. In practice this means that anyone can specify the `mTOFT` address as the receiver and initiate Stargate Swap from any chain to the `mTOFT` contract.

In conjunction with the first issue, this opens up the possibility of a DoS attack.

Let's imagine the following scenario: 

- Rebalancing operation needs to be performed between `mTOFT` on Ethereum and Avalanche that hold `USDC` as the underlying token.
- Rebalancing is initiated from Ethereum but the `sgReceive` on Avalanche fails and 1000 USDCs are sitting on `mTOFT` contract on Avalanche.
- A griever noticed this and initiated Stargate swap from Ethereum to Avalanche for 1 USDT specifying the `mTOFT` contract as the receiver.
- This is successful and now `mTOFT` has 1 `USDT` but 999 `USDC` as the griever's transaction has called the `sgRecieve` function that pushed 1 USDC to the `TOFTVault`. 
- As a consequence, the `clearCachedSwap` function fails because it tries to transfer the original 1000 USDC. 

```solidity
    function sgReceive(uint16, bytes memory, uint256, address, uint256 amountLD, bytes memory) external payable {
        if (msg.sender != _stargateRouter) revert mTOFT_NotAuthorized();

        if (erc20 == address(0)) {
            vault.depositNative{value: amountLD}();
        } else {
>>>>>            IERC20(erc20).safeTransfer(address(vault), amountLD); // amountLD is the original 1000 USDC
        }
    }
```
- The only solution here is to manually transfer that 1 USDC to the `mTOFT` contract and try calling the `clearCachedSwap` again.
- The griever can repeat this process multiple times.


## Impact
Hardcoding the `dstGasCall` to 0 in conjuction with not checking the `sgReceive` parameters opens up the possibility of a long-term DoS attack.  

## Code Snippet
- https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/mTOFT.sol#L326
- https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/Balancer.sol#L316

## Tool used

Manual Review

## Recommendation

The `dstGasForCall` shouldn't be hardcoded to 0. It should be a configurable value that is set by the admin of the `Balancer` contract. 

Take into account that this value will be different for different chains.

For instance, Arbitrum has a different gas model than Ethereum due to its specific precompiles: https://docs.arbitrum.io/arbos/gas.
<img width="865" alt="Screenshot 2024-03-13 at 14 52 09" src="https://github.com/sherlock-audit/2024-02-tapioca-windhustler/assets/38017754/874dbe44-6641-47a2-b41b-6c55e2a93a3c">

The recommended solution is:

```diff
 contract Balancer is Ownable {
     using SafeERC20 for IERC20;

+    mapping(uint16 => uint256) internal sgReceiveGas;

+    function setSgReceiveGas(uint16 eid, uint256 gas) external onlyOwner {
+        sgReceiveGas[eid] = gas;
+    }
+
+    function getSgReceiveGas(uint16 eid) internal view returns (uint256) {
+        uint256 gas = sgReceiveGas[eid];
+        if (gas == 0) revert();
+        return gas;
+    }
+
-    IStargateRouterBase.lzTxObj({dstGasForCall: 0, dstNativeAmount: 0, dstNativeAddr: "0x0"}),
+    IStargateRouterBase.lzTxObj({dstGasForCall: getSgReceiveGas(_dstChainId), dstNativeAmount: 0, dstNativeAddr: "0x0"}),
```
