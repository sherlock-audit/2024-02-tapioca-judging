Rural Amethyst Tapir

medium

# Composing approval with other messages is subject to DoS

## Summary

`TOFT::sendPacket` function allows the caller to specify multiple messages that are executed on the destination chain. 
On the receiving side the `lzCompose` function in `TOFT` contract can be DoS-ed by front-running the approval message and causing the `lzCompose` to revert. 
As `lzCompose` is supposed to process several messages, this results in lost fee paid on the sending chain for executing the subsequent messages and any value or gas airdropped to the contract.

## Vulnerability Detail
[`TOFT::sendPacket`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/TOFT.sol#L182) allows the caller to specify arbitrary `_composeMsg`.  It can be a single message or multiple composed messages. 

```solidity
>>>>    function sendPacket(LZSendParam calldata _lzSendParam, bytes calldata _composeMsg)
        public
        payable
        whenNotPaused // @audit Pausing is not implemented yet.
        returns (MessagingReceipt memory msgReceipt, OFTReceipt memory oftReceipt)
    {
        (msgReceipt, oftReceipt) = abi.decode(
            _executeModule(
                uint8(ITOFT.Module.TOFTSender),
>>>>               abi.encodeCall(TapiocaOmnichainSender.sendPacket, (_lzSendParam, _composeMsg)),
                false
            ),
            (MessagingReceipt, OFTReceipt)
        );
    }

```
If we observe the logic inside the `lzCompose`:

```solidity
    function _lzCompose(address srcChainSender_, bytes32 _guid, bytes memory oftComposeMsg_) internal {
        // Decode OFT compose message.
>>>>        (uint16 msgType_,,, bytes memory tapComposeMsg_, bytes memory nextMsg_) =
>>>>            TapiocaOmnichainEngineCodec.decodeToeComposeMsg(oftComposeMsg_);

        // Call Permits/approvals if the msg type is a permit/approval.
        // If the msg type is not a permit/approval, it will call the other receivers.
        if (msgType_ == MSG_REMOTE_TRANSFER) {
            _remoteTransferReceiver(srcChainSender_, tapComposeMsg_);
        } else if (!_extExec(msgType_, tapComposeMsg_)) {
            // Check if the TOE extender is set and the msg type is valid. If so, call the TOE extender to handle msg.
            if (
                address(tapiocaOmnichainReceiveExtender) != address(0)
                    && tapiocaOmnichainReceiveExtender.isMsgTypeValid(msgType_)
            ) {
                bytes memory callData = abi.encodeWithSelector(
                    ITapiocaOmnichainReceiveExtender.toeComposeReceiver.selector,
                    msgType_,
                    srcChainSender_,
                    tapComposeMsg_
                );
                (bool success, bytes memory returnData) =
                    address(tapiocaOmnichainReceiveExtender).delegatecall(callData);
                if (!success) {
                    revert(_getTOEExtenderRevertMsg(returnData));
                }
            } else {
                // If no TOE extender is set or msg type doesn't match extender, try to call the internal receiver.
                if (!_toeComposeReceiver(msgType_, srcChainSender_, tapComposeMsg_)) {
                    revert InvalidMsgType(msgType_);
                }
            }
        }

        emit ComposeReceived(msgType_, _guid, tapComposeMsg_);
>>>>        if (nextMsg_.length > 0) {
>>>>            _lzCompose(address(this), _guid, nextMsg_);
        }
    }
```

At the beginning of the function bytes memory `tapComposeMsg_` is the message being processed, while `bytes memory nextMsg_` are all the other messages. `lzCompose` will process all the messages until `nextMsg_` is empty.

A user might want to have his first message to grant approval, e.g. `_extExec` function call, while his second message might execute `BaseTOFTReceiver::_toeComposeReceiver` with `_msgType == MSG_YB_SEND_SGL_BORROW`. 

This is a problem as there is a clear DoS attack vector on granting any approvals. A griever can observe the permit message from the user and front-run the `lzCompose` call and submit the approval on the user's behalf. 

As permits use nonce it can't be replayed, which means if anyone front-runs the permit, the original permit will revert.
This means that `lzCompose` always reverts and all the gas and value to process the `BaseTOFTReceiver::_toeComposeReceiver` with `_msgType == MSG_YB_SEND_SGL_BORROW` is lost for the user. 

Permit based DoS attack is described in detail in the following article by Trust-Security: https://www.trust-security.xyz/post/permission-denied.

## Impact
When user is granting approvals and wants to execute any other message in the same `lzCompose` call, the attacker can deny the user from executing the other message by front-running the approval message and causing the `lzCompose` to revert.
The impact is lost fee paid on the sending chain for executing the subsequent messages and any value or gas airdropped to the contract. This is especially severe when the user wants to withdraw funds to another chain, as he needs to pay for that fee on the sending chain. 

## Code Snippet
- https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/TOFT.sol#L182


## Tool used

Manual Review

## Recommendation
`TOFT::sendPacket` should do extra checks to ensure if the message contains approvals, it should not allow packing several messages.
