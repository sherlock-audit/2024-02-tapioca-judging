Rural Amethyst Tapir

high

# All ETH can be stolen during rebalancing for `mTOFTs` that hold native

## Summary
Rebalancing of ETH transfers the ETH to the destination mTOFT without calling `sgRecieve` which leaves the ETH hanging inside the `mTOFT` contract. 
This can be exploited to steal all the ETH.

## Vulnerability Detail
Rebalancing of `mTOFTs` that hold native tokens is done through the `routerETH` contract inside the `Balancer.sol` contract. 
Here is the code snippet for the `routerETH` contract:

```solidity
## Balancer.sol

if (address(this).balance < _amount) revert ExceedsBalance();
        uint256 valueAmount = msg.value + _amount;
        routerETH.swapETH{value: valueAmount}(
            _dstChainId,
            payable(this),
            abi.encodePacked(connectedOFTs[_oft][_dstChainId].dstOft),
            _amount,
            _computeMinAmount(_amount, _slippage)
        );
```

The expected behaviour is ETH being received on the destination chain whereby `sgReceive` is called and ETH is deposited inside the `TOFTVault`.

```solidity
## mTOFT.sol

    function sgReceive(uint16, bytes memory, uint256, address, uint256 amountLD, bytes memory) external payable {
        if (msg.sender != _stargateRouter) revert mTOFT_NotAuthorized();

        if (erc20 == address(0)) {
            vault.depositNative{value: amountLD}();
        } else {
            IERC20(erc20).safeTransfer(address(vault), amountLD);
        }
    }
```

By taking a closer look at the logic inside the [`routerETH`](https://www.codeslaw.app/contracts/ethereum/0x150f94B44927F078737562f0fcF3C95c01Cc2376) contract we can see that the transfer is called with an empty payload:
```solidity
    // compose stargate to swap ETH on the source to ETH on the destination
    function swapETH(
        uint16 _dstChainId,                         // destination Stargate chainId
        address payable _refundAddress,             // refund additional messageFee to this address
        bytes calldata _toAddress,                  // the receiver of the destination ETH
        uint256 _amountLD,                          // the amount, in Local Decimals, to be swapped
        uint256 _minAmountLD                        // the minimum amount accepted out on destination
    ) external payable {
        require(msg.value > _amountLD, "Stargate: msg.value must be > _amountLD");

        // wrap the ETH into WETH
        IStargateEthVault(stargateEthVault).deposit{value: _amountLD}();
        IStargateEthVault(stargateEthVault).approve(address(stargateRouter), _amountLD);

        // messageFee is the remainder of the msg.value after wrap
        uint256 messageFee = msg.value - _amountLD;

        // compose a stargate swap() using the WETH that was just wrapped
        stargateRouter.swap{value: messageFee}(
            _dstChainId,                        // destination Stargate chainId
            poolId,                             // WETH Stargate poolId on source
            poolId,                             // WETH Stargate poolId on destination
            _refundAddress,                     // message refund address if overpaid
            _amountLD,                          // the amount in Local Decimals to swap()
            _minAmountLD,                       // the minimum amount swap()er would allow to get out (ie: slippage)
            IStargateRouter.lzTxObj(0, 0, "0x"),
            _toAddress,                         // address on destination to send to
>>>>>>      bytes("")                           // empty payload, since sending to EOA
        );
    }
```

Notice the comment:

> empty payload, since sending to EOA

So `routerETH` after depositing ETH in `StargateEthVault` calls the regular `StargateRouter` but with an empty payload. 

Next, let's see how the receiving logic works.

As Stargate is just another application built on top of LayerZero the receiving starts inside the [`Bridge::lzReceive`](https://github.com/stargate-protocol/stargate/blob/5f0dfd290d8290678b933b64f31d119b3c4b4e6a/contracts/Bridge.sol#L58) function.
As the type of transfer is `TYPE_SWAP_REMOTE` the `router::swapRemote` is called:

```solidity
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
            uint256 dstGasForCall,
            Pool.CreditObj memory c,
            Pool.SwapObj memory s,
            bytes memory to,
            bytes memory payload
        ) = abi.decode(_payload, (uint8, uint256, uint256, uint256, Pool.CreditObj, Pool.SwapObj, bytes, bytes));
        address toAddress;
        assembly {
            toAddress := mload(add(to, 20))
        }
        router.creditChainPath(_srcChainId, srcPoolId, dstPoolId, c);
>>>>>>  router.swapRemote(_srcChainId, _srcAddress, _nonce, srcPoolId, dstPoolId, dstGasForCall, toAddress, s, payload);
```

[`Router:swapRemote`](https://github.com/stargate-protocol/stargate/blob/5f0dfd290d8290678b933b64f31d119b3c4b4e6a/contracts/Router.sol#L390-#L425) has two responsibilities:
- First it calls `pool::swapRemote` that transfers the actual tokens to the destination address. In this case this is the `mTOFT` contract. 
- Second it will call `IStargateReceiver(mTOFTAddress)::sgReceive` but only if the payload is not empty.

```solidity
 function _swapRemote(
    uint16 _srcChainId,
    bytes memory _srcAddress,
    uint256 _nonce,
    uint256 _srcPoolId,
    uint256 _dstPoolId,
    uint256 _dstGasForCall,
    address _to,
    Pool.SwapObj memory _s,
    bytes memory _payload
) internal {
    Pool pool = _getPool(_dstPoolId);
    // first try catch the swap remote
    try pool.swapRemote(_srcChainId, _srcPoolId, _to, _s) returns (uint256 amountLD) {
>>>>>>   if (_payload.length > 0) {
            // then try catch the external contract call
>>>>>>      try IStargateReceiver(_to).sgReceive{gas: _dstGasForCall}(_srcChainId, _srcAddress, _nonce, pool.token(), amountLD, _payload) {
                // do nothing
            } catch (bytes memory reason) {
                cachedSwapLookup[_srcChainId][_srcAddress][_nonce] = CachedSwap(pool.token(), amountLD, _to, _payload);
                emit CachedSwapSaved(_srcChainId, _srcAddress, _nonce, pool.token(), amountLD, _to, _payload, reason);
            }
        }
    } catch {
        revertLookup[_srcChainId][_srcAddress][_nonce] = abi.encode(
            TYPE_SWAP_REMOTE_RETRY,
            _srcPoolId,
            _dstPoolId,
            _dstGasForCall,
            _to,
            _s,
            _payload
        );
        emit Revert(TYPE_SWAP_REMOTE_RETRY, _srcChainId, _srcAddress, _nonce);
    }
}
```

As payload is empty in case of using the `routerETH` contract the `sgReceive` function is never called. This means that the ETH is left sitting inside the `mTOFT` contract. 

There are several ways of stealing the balance of `mTOFT`. 
An attacker can use the [`mTOFT::sendPacket`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/mTOFT.sol#L234) function and utilize the `lzNativeGasDrop` option to airdrop the balance of mTOFT to attacker's address on the destination chain: https://docs.layerzero.network/contracts/options#lznativedrop-option

```solidity
## TapiocaOmnichainSender.sol

 function sendPacket(LZSendParam calldata _lzSendParam, bytes calldata _composeMsg)
        external
        payable
        returns (MessagingReceipt memory msgReceipt, OFTReceipt memory oftReceipt)
    {
        // @dev Applies the token transfers regarding this send() operation.
        // - amountDebitedLD is the amount in local decimals that was ACTUALLY debited from the sender.
        // - amountToCreditLD is the amount in local decimals that will be credited to the recipient on the remote OFT instance.
        (uint256 amountDebitedLD, uint256 amountToCreditLD) =
            _debit(_lzSendParam.sendParam.amountLD, _lzSendParam.sendParam.minAmountLD, _lzSendParam.sendParam.dstEid);

        // @dev Builds the options and OFT message to quote in the endpoint.
        (bytes memory message, bytes memory options) =
            _buildOFTMsgAndOptions(_lzSendParam.sendParam, _lzSendParam.extraOptions, _composeMsg, amountToCreditLD);

        // @dev Sends the message to the LayerZero endpoint and returns the LayerZero msg receipt.
        msgReceipt =
            _lzSend(_lzSendParam.sendParam.dstEid, message, options, _lzSendParam.fee, _lzSendParam.refundAddress);
        // @dev Formulate the OFT receipt.
        oftReceipt = OFTReceipt(amountDebitedLD, amountToCreditLD);

        emit OFTSent(msgReceipt.guid, _lzSendParam.sendParam.dstEid, msg.sender, amountDebitedLD);
    }
```

All he has to do is specify the option type `lzNativeDrop` inside the `_lsSendParams.extraOptions` and the cost of calling `_lzSend` plus the airdrop amount will be paid out from the balance of `mTOFT`.

As this is a complete theft of the rebalanced amount I'm rating this as a critical vulnerability.


## Impact
All ETH can be stolen during rebalancing for mTOFTs that hold native tokens.

## Code Snippet
- https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/Balancer.sol#L269-#L279


## Tool used

Manual Review

## Recommendation
One way to fix this is use the alternative `RouterETH.sol` contract available from Stargate that allows for a payload to be sent. 
It is denoted as `*RouterETH.sol` in the Stargate documentation: https://stargateprotocol.gitbook.io/stargate/developers/contract-addresses/mainnet
This router has the `swapETHAndCall` interface:

```solidity
function swapETHAndCall(
        uint16 _dstChainId, // destination Stargate chainId
        address payable _refundAddress, // refund additional messageFee to this address
        bytes calldata _toAddress, // the receiver of the destination ETH
        SwapAmount memory _swapAmount, // the amount and the minimum swap amount
        IStargateRouter.lzTxObj memory _lzTxParams, // the LZ tx params
        bytes calldata _payload // the payload to send to the destination
    ) external payable {
```

The contract on Ethereum can be found at: https://www.codeslaw.app/contracts/ethereum/0xb1b2eeF380f21747944f46d28f683cD1FBB4d03c. 
And the Stargate docs specify its deployment address on all the chains where ETH is supported: https://stargateprotocol.gitbook.io/stargate/developers/contract-addresses/mainnet
