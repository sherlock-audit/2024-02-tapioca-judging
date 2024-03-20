Vast Bronze Salamander

medium

# mTOFTReceiver MSG_XCHAIN_LEND_XCHAIN_LOCK unable to execute

## Summary
In `mTOFTReceiver._toftCustomComposeReceiver(uint16 _msgType)`
If `_msgType` is processed normally, the method must return `true`, if it returns `false`, it will trigger `revert InvalidMsgType()`
But when `_msgType == MSG_XCHAIN_LEND_XCHAIN_LOCK` is executed normally, it does not correctly return true
This causes this type of execution to always fail
## Vulnerability Detail
The main execution order of `_lzCompose()` is as follows:
1. If msgType_ == MSG_REMOTE_TRANSFER, execute `_remoteTransferReceiver()`
2. Otherwise, execute `_extExec(msgType_, tapComposeMsg_)`
3. Otherwise, execute `tapiocaOmnichainReceiveExtender`
4. Otherwise, execute `_toeComposeReceiver()`
5. If the 4th step `_toeComposeReceiver()` returns false, it is considered that the type cannot be found, and `revert InvalidMsgType(msgType_);` is triggered

the code as follows：
```solidity
    function _lzCompose(address srcChainSender_, bytes32 _guid, bytes memory oftComposeMsg_) internal {
        // Decode OFT compose message.
        (uint16 msgType_,,, bytes memory tapComposeMsg_, bytes memory nextMsg_) =
            TapiocaOmnichainEngineCodec.decodeToeComposeMsg(oftComposeMsg_);

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
@>                  revert InvalidMsgType(msgType_);
                }
            }
        }
```

The implementation of `mTOFTReceiver._toeComposeReceiver()` is as follows:

```solidity
contract mTOFTReceiver is BaseTOFTReceiver {
    constructor(TOFTInitStruct memory _data) BaseTOFTReceiver(_data) {}

    function _toftCustomComposeReceiver(uint16 _msgType, address, bytes memory _toeComposeMsg)
        internal
        override
        returns (bool success)
    {
        if (_msgType == MSG_LEVERAGE_UP) { //@check
            _executeModule(
                uint8(ITOFT.Module.TOFTMarketReceiver),
                abi.encodeWithSelector(TOFTMarketReceiverModule.leverageUpReceiver.selector, _toeComposeMsg),
                false
            );
            return true;
        } else if (_msgType == MSG_XCHAIN_LEND_XCHAIN_LOCK) { //@check
            _executeModule(
                uint8(ITOFT.Module.TOFTOptionsReceiver),
                abi.encodeWithSelector(
                    TOFTOptionsReceiverModule.mintLendXChainSGLXChainLockAndParticipateReceiver.selector, _toeComposeMsg
                ),
                false
            );
@>      //@audit miss return true
        } else {
            return false;
        }
    }
}
```

As mentioned above, because `_msgType == MSG_XCHAIN_LEND_XCHAIN_LOCK` does not return `true`, it always triggers `revert InvalidMsgType(msgType_);`


## Impact
`_msgType == MSG_XCHAIN_LEND_XCHAIN_LOCK`
`TOFTOptionsReceiver.mintLendXChainSGLXChainLockAndParticipateReceiver()` unable to execute successfully
## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/mTOFTReceiver.sol#L36-L43
## Tool used

Manual Review

## Recommendation

```diff
contract mTOFTReceiver is BaseTOFTReceiver {
    constructor(TOFTInitStruct memory _data) BaseTOFTReceiver(_data) {}

    function _toftCustomComposeReceiver(uint16 _msgType, address, bytes memory _toeComposeMsg)
        internal
        override
        returns (bool success)
    {
        if (_msgType == MSG_LEVERAGE_UP) { //@check
            _executeModule(
                uint8(ITOFT.Module.TOFTMarketReceiver),
                abi.encodeWithSelector(TOFTMarketReceiverModule.leverageUpReceiver.selector, _toeComposeMsg),
                false
            );
            return true;
        } else if (_msgType == MSG_XCHAIN_LEND_XCHAIN_LOCK) { //@check
            _executeModule(
                uint8(ITOFT.Module.TOFTOptionsReceiver),
                abi.encodeWithSelector(
                    TOFTOptionsReceiverModule.mintLendXChainSGLXChainLockAndParticipateReceiver.selector, _toeComposeMsg
                ),
                false
            );
+           return true;
        } else {
            return false;
        }
    }
}
```
