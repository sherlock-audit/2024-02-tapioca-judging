Zealous Pineapple Duck

medium

# Any excess native token funds sent to TOFTGenericReceiverModule's `receiveWithParamsReceiver()` can be immediately extracted by anyone via back-running

## Summary

`msg.value - msg_.amount` can be stolen right after the `receiveWithParamsReceiver()` call each time it's big enough to cover attacker's gas costs.

## Vulnerability Detail

Excess funds occurrences can be expected in general in user facing operations. There also exists a share of cases when `msg.value - msg_.amount` isn't just a residue, but is material, being a result of user operational mistake. Attacker can setup a bot, automatically tracking all such events, immediately extracting these funds via back-running whenever the expected result exceeds gas costs.

## Impact

Native tokens are a free grab from contract balance and `msg.value - msg_.amount` can be instantly lost this way to back-running. The prerequisite is call having more than `msg_.amount` attached, so the probability is low. Funds stealing impact has high severity.

Likelihood: Low + Impact: High = Severity: Medium.

## Code Snippet

In addition `msg.value - msg_.amount` excess being stolen, when `msg_.unwrap == false` the `msg.value` attached to a call will be lost in a similar way:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTGenericReceiverModule.sol#L47-L67

```solidity
    function receiveWithParamsReceiver(address srcChainSender, bytes memory _data) public payable {
        SendParamsMsg memory msg_ = TOFTMsgCodec.decodeSendParamsMsg(_data);

        msg_.amount = _toLD(msg_.amount.toUint64());

>>      if (msg_.unwrap) { /// @audit what if not?
            ITOFT tOFT = ITOFT(address(this));
            address toftERC20 = tOFT.erc20();

            /// @dev xChain owner needs to have approved dst srcChain `sendPacket()` msg.sender in a previous composedMsg. Or be the same address.
            _internalTransferWithAllowance(msg_.receiver, srcChainSender, msg_.amount);
            tOFT.unwrap(address(this), msg_.amount);

            if (toftERC20 != address(0)) {
                IERC20(toftERC20).safeTransfer(msg_.receiver, msg_.amount);
            } else {
>>              (bool sent,) = msg_.receiver.call{value: msg_.amount}("");
                if (!sent) revert TOFTGenericReceiverModule_TransferFailed();
            }
        }
    }
```

## Tool used

Manual Review

## Recommendation

Consider checking for excess and zero values, e.g.:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTGenericReceiverModule.sol#L47-L67

```diff
    function receiveWithParamsReceiver(address srcChainSender, bytes memory _data) public payable {
        SendParamsMsg memory msg_ = TOFTMsgCodec.decodeSendParamsMsg(_data);

        msg_.amount = _toLD(msg_.amount.toUint64());

        if (msg_.unwrap) {
            ITOFT tOFT = ITOFT(address(this));
            address toftERC20 = tOFT.erc20();

            /// @dev xChain owner needs to have approved dst srcChain `sendPacket()` msg.sender in a previous composedMsg. Or be the same address.
            _internalTransferWithAllowance(msg_.receiver, srcChainSender, msg_.amount);
            tOFT.unwrap(address(this), msg_.amount);

            if (toftERC20 != address(0)) {
                IERC20(toftERC20).safeTransfer(msg_.receiver, msg_.amount);
            } else {
+               if (msg.value != msg_.amount) revert TOFTGenericReceiverModule_AmountMismatch();
                (bool sent,) = msg_.receiver.call{value: msg_.amount}("");
                if (!sent) revert TOFTGenericReceiverModule_TransferFailed();
            }
-       }
+       } else {
+           if (msg.value > 0) revert TOFTGenericReceiverModule_AmountMismatch();
+       }
    }
}
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTGenericReceiverModule.sol#L34

```diff
    error TOFTGenericReceiverModule_TransferFailed();
+   error TOFTGenericReceiverModule_AmountMismatch();
```