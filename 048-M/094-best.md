Vast Bronze Salamander

medium

# Balancer using safeApprove may lead to revert.

## Summary
When executing `Balancer._routerSwap()`, the `oz` `safeApprove` function is used to set an allowance. 
Due to the presence of the `convertRate` in the `router`, `Balancer._routerSwap()` rounds down the incoming quantity. 
This behavior may result in the allowance not being fully use, causing a subsequent execution of `oz.safeApprove()` to revert.

## Vulnerability Detail
The code snippet for `Balancer._routerSwap()` is as follows:


```solidity
    function _routerSwap(
        uint16 _dstChainId,
        uint256 _srcPoolId,
        uint256 _dstPoolId,
        uint256 _amount,
        uint256 _slippage,
        address payable _oft,
        address _erc20
    ) private {
        bytes memory _dst = abi.encodePacked(connectedOFTs[_oft][_dstChainId].dstOft);
@>      IERC20(_erc20).safeApprove(address(router), _amount);
        router.swap{value: msg.value}(
            _dstChainId,
            _srcPoolId,
            _dstPoolId,
            payable(this),
            _amount,
            _computeMinAmount(_amount, _slippage),
            IStargateRouterBase.lzTxObj({dstGasForCall: 0, dstNativeAmount: 0, dstNativeAddr: "0x0"}),
            _dst,
            "0x"
        );
    }
```
In the above code, `SafeERC20.safeApprove()` from the `oz` library is used, but the allowance is not cleared afterward. Consequently, if the current allowance is not fully use during this transaction, a subsequent execution of `SafeERC20.safeApprove()` will revert.

Is it guaranteed that `router.swap()` will fully use the allowance?
Not necessarily. Due to the presence of `convertRate` in the implementation code, the `router` rounds down the amount, potentially leaving a remainder in the allowance.
DAI pool convertRate = 1e12
DAI pool: https://etherscan.io/address/0x0Faf1d2d3CED330824de3B8200fc8dc6E397850d#readContract

router codes:
https://etherscan.io/address/0x8731d54E9D02c286767d56ac03e8037C07e01e98#code
```solidity
    function swap(
        uint16 _dstChainId,
        uint256 _srcPoolId,
        uint256 _dstPoolId,
        address payable _refundAddress,
        uint256 _amountLD,
        uint256 _minAmountLD,
        lzTxObj memory _lzTxParams,
        bytes calldata _to,
        bytes calldata _payload
    ) external payable override nonReentrant {
        require(_amountLD > 0, "Stargate: cannot swap 0");
        require(_refundAddress != address(0x0), "Stargate: _refundAddress cannot be 0x0");
        Pool.SwapObj memory s;
        Pool.CreditObj memory c;
        {
            Pool pool = _getPool(_srcPoolId);
            {
@>              uint256 convertRate = pool.convertRate();
@>              _amountLD = _amountLD.div(convertRate).mul(convertRate);
            }

            s = pool.swap(_dstChainId, _dstPoolId, msg.sender, _amountLD, _minAmountLD, true);
            _safeTransferFrom(pool.token(), msg.sender, address(pool), _amountLD);
            c = pool.sendCredits(_dstChainId, _dstPoolId);
        }
        bridge.swap{value: msg.value}(_dstChainId, _srcPoolId, _dstPoolId, _refundAddress, c, s, _lzTxParams, _to, _payload);
    }
```

## Impact

Unused allowance may lead to failure in subsequent `_routerSwap()` executions.

## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/Balancer.sol#L308
## Tool used

Manual Review

## Recommendation
```diff
    function _routerSwap(
        uint16 _dstChainId,
        uint256 _srcPoolId,
        uint256 _dstPoolId,
        uint256 _amount,
        uint256 _slippage,
        address payable _oft,
        address _erc20
    ) private {
        bytes memory _dst = abi.encodePacked(connectedOFTs[_oft][_dstChainId].dstOft);
        IERC20(_erc20).safeApprove(address(router), _amount);
        router.swap{value: msg.value}(
            _dstChainId,
            _srcPoolId,
            _dstPoolId,
            payable(this),
            _amount,
            _computeMinAmount(_amount, _slippage),
            IStargateRouterBase.lzTxObj({dstGasForCall: 0, dstNativeAmount: 0, dstNativeAddr: "0x0"}),
            _dst,
            "0x"
        );
+       IERC20(_erc20).safeApprove(address(router), 0);
```