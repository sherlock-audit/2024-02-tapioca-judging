Zealous Pineapple Duck

high

# TOFTOptionsReceiverModule will have the user lose the whole output TAP when requested to exercise all eligible options

## Summary

TOFTOptionsReceiverModule's `exerciseOptionsReceiver()` will execute successfully, but lose (freeze permanently) all the output TAP amount of the user if being run with zero TAP amount (`_options.tapAmount`), which is valid use case of TapiocaOptionBroker's `exerciseOption()`, corresponding to the full position exercise.

## Vulnerability Detail

Specifying zero tap amount is a usual workflow of TapiocaOptionBroker's `exerciseOption()`, meaning that the whole eligible option position should be exercised. It's arguably the most used way to interact with `exerciseOption()` since slicing the exercise doesn't provide any additional benefits, but increases the operational and gas costs.

`exerciseOptionsReceiver()` will not revert when run with `_options.tapAmount = 0`, it will exercise the full position, but send nothing to the user: the whole TAP amount received will be left with the contract, being permanently frozen there as there is no way to rescue it.

## Impact

The probability of having `exerciseOptionsReceiver()` run with `_options.tapAmount = 0` can be estimated as medium. The impact of user losing the whole position TAP proceedings, being permanently frozen with the contract, has high severity.

Likelihood: Medium + Impact: High = Severity: High.

## Code Snippet

`_options.tapAmount == 0` is an allowed state for `exerciseOptionsReceiver()`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L142-L181

```solidity
    function exerciseOptionsReceiver(address srcChainSender, bytes memory _data) public payable {
        // Decode received message.
        ExerciseOptionsMsg memory msg_ = TOFTMsgCodec.decodeExerciseOptionsMsg(_data);

        _checkWhitelistStatus(msg_.optionsData.target);
        _checkWhitelistStatus(OFTMsgCodec.bytes32ToAddress(msg_.lzSendParams.sendParam.to));

        {
            // _data declared for visibility.
            IExerciseOptionsData memory _options = msg_.optionsData;
>>          _options.tapAmount = _toLD(_options.tapAmount.toUint64());
            _options.paymentTokenAmount = _toLD(_options.paymentTokenAmount.toUint64());

            // @dev retrieve paymentToken amount
            _internalTransferWithAllowance(_options.from, srcChainSender, _options.paymentTokenAmount);

            /// Does this: _approve(address(this), _options.target, _options.paymentTokenAmount);
            pearlmit.approve(
                address(this), 0, _options.target, uint200(_options.paymentTokenAmount), uint48(block.timestamp + 1)
            ); // Atomic approval
            address(this).safeApprove(address(pearlmit), _options.paymentTokenAmount);

            /// @dev call exerciseOption() with address(this) as the payment token
            uint256 bBefore = balanceOf(address(this));
            ITapiocaOptionBroker(_options.target).exerciseOption(
                _options.oTAPTokenID,
                address(this), //payment token
>>              _options.tapAmount
            );
            address(this).safeApprove(address(pearlmit), 0); // Clear approval
            uint256 bAfter = balanceOf(address(this));

            // Refund if less was used.
            if (bBefore > bAfter) {
                uint256 diff = bBefore - bAfter;
                if (diff < _options.paymentTokenAmount) {
                    IERC20(address(this)).safeTransfer(_options.from, _options.paymentTokenAmount - diff);
                }
            }
        }
```


It corresponds to a situation of exercising for the whole eligible TAP amount in TapiocaOptionBroker's `exerciseOption()`:

https://github.com/Tapioca-DAO/tap-token/blob/main/contracts/options/TapiocaOptionBroker.sol#L390-L402

```solidity
        uint256 eligibleTapAmount = muldiv(tOLPLockPosition.ybShares, gaugeTotalForEpoch, netAmount);
        eligibleTapAmount -= oTAPCalls[_oTAPTokenID][cachedEpoch]; // Subtract already exercised amount
        if (eligibleTapAmount < _tapAmount) revert TooHigh();

>>      uint256 chosenAmount = _tapAmount == 0 ? eligibleTapAmount : _tapAmount;
        if (chosenAmount < 1e18) revert TooLow();
        oTAPCalls[_oTAPTokenID][cachedEpoch] += chosenAmount; // Adds up exercised amount to current epoch

        // Finalize the deal
>>      _processOTCDeal(_paymentToken, paymentTokenOracle, chosenAmount, oTAPPosition.discount);

        emit ExerciseOption(cachedEpoch, msg.sender, _paymentToken, _oTAPTokenID, chosenAmount);
    }

```

But `exerciseOptionsReceiver()` will send out nothing in this case, the whole TAP amount received will be left with the contract instead of being forwarded to the user:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L183-L207

```solidity
        {
            // _data declared for visibility.
            IExerciseOptionsData memory _options = msg_.optionsData;
            SendParam memory _send = msg_.lzSendParams.sendParam;

            address tapOft = ITapiocaOptionBroker(_options.target).tapOFT();
            if (msg_.withdrawOnOtherChain) {
                /// @dev determine the right amount to send back to source
>>              uint256 amountToSend = _send.amountLD > _options.tapAmount ? _options.tapAmount : _send.amountLD;
                if (_send.minAmountLD > amountToSend) {
                    _send.minAmountLD = amountToSend;
                }

                // Sends to source and preserve source `msg.sender` (`from` in this case).
                _sendPacket(msg_.lzSendParams, msg_.composeMsg, _options.from);

                // Refund extra amounts
                if (_options.tapAmount - amountToSend > 0) {
                    IERC20(tapOft).safeTransfer(_options.from, _options.tapAmount - amountToSend);
                }
            } else {
                //send on this chain
>>              IERC20(tapOft).safeTransfer(_options.from, _options.tapAmount);
            }
        }
```

I.e. it will be `amountToSend = _send.minAmountLD = 0` when `msg_.withdrawOnOtherChain == true` and just `_options.tapAmount = 0` otherwise, so all the TAP proceedings will stay with the contract. Since there is no possibility to receive TAP funds out of the contract in excess to the `oTAPTokenID` eligible amount, which becomes zero after exercise, these TAP proceedings will be permanently frozen on the contract balance.

## Tool used

Manual Review

## Recommendation

Consider either forbidding zero `_options.tapAmount` in `exerciseOptionsReceiver()` or adding `nonReentrant` modifier to it, tracking TAP token balance and sending out the realized balance difference from TapiocaOptionBroker's `exerciseOption()` operation to the user instead of relying on `_options.tapAmount`.