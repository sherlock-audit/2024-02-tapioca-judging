Fast Carrot Turtle

medium

# Incorrect `tapOft` Amounts Will Be Sent to Desired Chains on Certain Conditions

## Summary
[TOFTOptionsReceiverModule::exerciseOptionsReceiver](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L142-L209) module, is responsible for facilitating users' token exercises between `mTOFT` and `tapOFT` tokens across different chains. 
In a `msg-type` where the user wishes to receive the `tapOFT` tokens on a different chain, the module attempts to ensure the amount sent to the user on the desired chain, aligns with the received tap amount in the current chain. However, a flaw exists where the computed amount to send is not updated in the send parameters, resulting in incorrect token transfer.
## Vulnerability Detail

[TOFTOptionsReceiverModule::exerciseOptionsReceiver](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L142-L209) module is a module that enables users to exercise their `mTOFT` tokens for a given amount of `tapOFT` option tokens.

When the user wishes to withdraw these `tapOft` tokens on a different chain, the [withdrawOnOtherChain](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L189) param will be set to true. For this composed call type, the contract attempts to ensure the amount to send to the user on the other chain isn't more than the received `tap amount`, by doing this:
```solidity
           uint256 amountToSend = _send.amountLD > _options.tapAmount ? _options.tapAmount : _send.amountLD;
                if (_send.minAmountLD > amountToSend) {
                    _send.minAmountLD = amountToSend;
                }
```
The issue here is that, the computed amount to send, is never updated in the `lsSendParams.sendParam`, the current code still goes on to send the packet to the destination chain with the default input amount:
```solidity
            if (msg_.withdrawOnOtherChain) {
                /// @dev determine the right amount to send back to source
                uint256 amountToSend = _send.amountLD > _options.tapAmount ? _options.tapAmount : _send.amountLD;
                if (_send.minAmountLD > amountToSend) {
                    _send.minAmountLD = amountToSend;
                }


                // Sends to source and preserve source `msg.sender` (`from` in this case).
                _sendPacket(msg_.lzSendParams, msg_.composeMsg, _options.from);


                // Refund extra amounts
                if (_options.tapAmount - amountToSend > 0) {
                    IERC20(tapOft).safeTransfer(_options.from, _options.tapAmount - amountToSend);
                }
```
+ To Illustrate:

assuming send `amountLD` = 100
and the user is to receive a tap amount of = 80
since `amountLD` is greater than tap amount, the amount to send should be 80, i.e. `msg_.lzSendParams.sendParam.amountLD` = 80
The current code goes on to send the default 100 to the user, when the user is only entitled to 80


## Impact
The user will always receive an incorrect amount of `tapOFT` in the desired chain whenever `amountLD` is greater than `tapAmount`
## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L142-L209
## Tool used

Manual Review

## Recommendation
update the lz send param `amountLD` to the new computed `amountToSend` before sending the packet
+ I.e :
```solidity
msg_.lzSendParams.sendParam.amountLD = amountToSend;
```
Note that the issue should also be fixed in Tapioca-Bar as well

https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/usdo/modules/UsdoOptionReceiverModule.sol#L113-L118

