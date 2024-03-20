Fast Carrot Turtle

medium

# Any Oft Token Holder Can be Forcefully Unwrapped

## Summary
`TOFTGenericReceiverModule` Module is intended to be called down [lzCompose](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/gitmodule/tapioca-periph/contracts/tapiocaOmnichainEngine/TapiocaOmnichainReceiver.sol#L120-L140) function execution, but can currently be abused, by calling this module directly via `mTOFT::executeModule` function, to forcefully unwrap any `mToft` token holder.
## Vulnerability Detail
The [TOFTGenericReceiverModule::receiveWithParamsReceiver](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/modules/TOFTGenericReceiverModule.sol#L47-L67) function is intended to be called via a composed message. For example:
A user does a cross-chain transfer from chain A to chain B, usually, the user will receive the equivalent of the burned oft token amount in chain A on chain B, as chain B's oft tokens, with `TOFTGenericReceiverModule` module, the user can add an added compose message to further unwrap this tokens via [TOFTGenericReceiverModule::receiveWithParamsReceiver](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/modules/TOFTGenericReceiverModule.sol#L47-L67) function for the underlying token.

+ What's the Issue?

The problem here is that, a griefer can forcefully unwrap any oft token holder by invoking [TOFTGenericReceiverModule::receiveWithParamsReceiver](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/modules/TOFTGenericReceiverModule.sol#L47-L67) function, via a call to [mTOFT::executeModule](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/mTOFT.sol#L198-L205) function, where the input module will be `ITOFT.Module.TOFTGenericReceiver` type, with the passed-in payload `SendParamzsMsg.receiver` address the same as the input `srcChainSender` address. 
Since the `srcChainSender` equals the receiver, the internal transfer succeeds, and the user oft tokens will be unwrapped for the underlying:
```solidity
    function _internalTransferWithAllowance(address _owner, address srcChainSender, uint256 _amount) internal {
        if (_owner != srcChainSender) {
            _spendAllowance(_owner, srcChainSender, _amount);
        }

        _transfer(_owner, address(this), _amount);
    }
```

## Impact
+ depletes liquidity: When a substantial amount of `mtoft` tokens have been forcefully unwrapped, unwrapping will be bricked, since there will no longer be any available liquidity.
+ A griefer can fail normal transfers of oft tokens by frontrunning the user txn and then forcefully unwrapping the sender off all his tokens
+ A griefer can fail cross-chain transfers by front-running cross-chain calls to forcefully unwrap the sender's oft tokens, then on the user's call execution it fails due to insufficient balance.


## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/modules/TOFTGenericReceiverModule.sol#L47-L67
## Tool used

Manual Review

## Recommendation
Since `receiveWithParamsReceiver` function is meant to be called when there is a compose message, down the execution in [lzCompose](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/gitmodule/tapioca-periph/contracts/tapiocaOmnichainEngine/TapiocaOmnichainReceiver.sol#L120-L140), the `msg.sender` here is expected to always be the endpoint address.

I will recommend whitelisting `lz endpoint`.

Add this function to `TOFTGenericReceiverModule`:
```solidity
    function _checkWhitelistStatus(address _addr) private view {
        if (_addr != address(0)) {
            if (!cluster.isWhitelisted(0, _addr)) {
                revert TOFTOptionsReceiverModule_NotAuthorized(_addr);
            }
        }
    }
```
Then add this check, in `TOFTGenericReceiverModule::receiveWithParamsReceiver` function:
```solidity

    function receiveWithParamsReceiver(address srcChainSender, bytes memory _data) public payable {
   +     _checkWhitelistStatus(msg.sender);
        SendParamzsMsg memory msg_ = TOFTMsgCodec.decodeSendParamsMsg(_data);


                         #############

}

```