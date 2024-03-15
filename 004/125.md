Skinny Wool Mallard

high

# Withdrawing to other chain when exercising options won’t work as expected, leading to DoS

## Summary

Withdrawing to another chain when exercising options will always fail because the implemented functionality does not bridge the tokens exercised in the option, and tries to perform a regular cross-chain call instead.

## Vulnerability Detail

Tapioca incorporates a **DAO Share Options** (**DSO)** program where users can lock USDO in order to obtain TAP tokens at a discounted price.

In order to exercise their options, users need to execute a compose call with a message type of `MSG_TAP_EXERCISE`, which will trigger the `UsdoOptionReceiverModule`'s `exerciseOptionsReceiver()` function.

When exercising their options, users can decide to bridge the obtained TAP tokens into another chain by setting the `msg_.withdrawOnOtherChain` to `true`:

```solidity
// UsdoOptionReceiverModule.sol

function exerciseOptionsReceiver(address srcChainSender, bytes memory _data) public payable {
        
        ... 
        
        ITapiocaOptionBroker(_options.target).exerciseOption(
            _options.oTAPTokenID,
            address(this), //payment token 
            _options.tapAmount 
        ); 
        
        ...
 
        address tapOft = ITapiocaOptionBroker(_options.target).tapOFT();
        if (msg_.withdrawOnOtherChain) {
			       ... 

            // Sends to source and preserve source `msg.sender` (`from` in this case).
            _sendPacket(msg_.lzSendParams, msg_.composeMsg, _options.from); 

            // Refund extra amounts
            if (_options.tapAmount - amountToSend > 0) {
                IERC20(tapOft).safeTransfer(_options.from, _options.tapAmount - amountToSend);
            }
        } else {
            //send on this chain
            IERC20(tapOft).safeTransfer(_options.from, _options.tapAmount);
        }
        }
    } 
```

As the code snippet shows, `exerciseOptionsReceiver()` will perform mainly 2 steps:

1. Exercise the option by calling `_options.target.exerciseOption()` . This  will make USDO tokens serving as a payment for the `tapOft` tokens be transferred from the user, and in exchange the corresponding option `tapOft` tokens will be transferred to the USDO contract so that they can later be transferred to the user.
2. TAP tokens will be sent to the user. This can be done in two ways:
    1. If the user doesn’t decide to bridge them (by leaving `msg_.withdrawOnOtherChain` as `false`), the `tapOft` tokens will simply be transferred to the `_options.from` address, succesfully exercising the option
    2. On the other hand, if the user decides to bridge the exercised option, the internal `_sendPacket()` function will be triggered, which will perform a call via LayerZero to the destination chain:
        
        ```solidity
        // UsdoOptionReceiverModule.sol
        
        function _sendPacket(LZSendParam memory _lzSendParam, bytes memory _composeMsg, address _srcChainSender)
                private
                returns (MessagingReceipt memory msgReceipt, OFTReceipt memory oftReceipt)
            {
                /// @dev Applies the token transfers regarding this send() operation.
                // - amountDebitedLD is the amount in local decimals that was ACTUALLY debited from the sender.
                // - amountToCreditLD is the amount in local decimals that will be credited to the recipient on the remote OFT instance.
                (uint256 amountDebitedLD, uint256 amountToCreditLD) =
                    _debit(_lzSendParam.sendParam.amountLD, _lzSendParam.sendParam.minAmountLD, _lzSendParam.sendParam.dstEid);
           
                /// @dev Builds the options and OFT message to quote in the endpoint.
                (bytes memory message, bytes memory options) = _buildOFTMsgAndOptionsMemory(
                    _lzSendParam.sendParam, _lzSendParam.extraOptions, _composeMsg, amountToCreditLD, _srcChainSender
                );
          
                /// @dev Sends the message to the LayerZero endpoint and returns the LayerZero msg receipt.
                msgReceipt =
                    _lzSend(_lzSendParam.sendParam.dstEid, message, options, _lzSendParam.fee, _lzSendParam.refundAddress);
                /// @dev Formulate the OFT receipt.
                oftReceipt = OFTReceipt(amountDebitedLD, amountToCreditLD);
        
                emit OFTSent(msgReceipt.guid, _lzSendParam.sendParam.dstEid, msg.sender, amountDebitedLD);
            }
        ```
        
    

The problem with the approach followed when users want to bridge the exercised options is that the contract will not actually bridge the exercised `tapOft` tokens by calling the `tapOft`'s `sendPacket()` function (which is the actual way by which the token can be transferred cross-chain). Instead, the contract calls `_sendPacket()` , a function that will try to  perform a USDO cross-chain call (instead of a `tapOft` cross-chain call). This will make the `_debit()` function inside `_sendPacket()` be executed, which will try to burn USDO tokens from the `msg.sender`:

```solidity
// OFT.sol 

function _debit(
        uint256 _amountLD, 
        uint256 _minAmountLD,
        uint32 _dstEid
    ) internal virtual override returns (uint256 amountSentLD, uint256 amountReceivedLD) {
        (amountSentLD, amountReceivedLD) = _debitView(_amountLD, _minAmountLD, _dstEid);
 
        // @dev In NON-default OFT, amountSentLD could be 100, with a 10% fee, the amountReceivedLD amount is 90,
        // therefore amountSentLD CAN differ from amountReceivedLD.
   
        // @dev Default OFT burns on src.
        _burn(msg.sender, amountSentLD);
    }
```

This leads to two possible outcomes:

1. `msg.sender` (the LayerZero endpoint) has enough `amountSentLD` of USDO tokens to be burnt. In this situation, USDO tokens will be incorrectly burnt from the user, leading to a loss of balance for him. After this, the burnt USDO tokens will be bridged.  This outcome greatly affect the user in two ways:
    1. USDO tokens are incorrectly burnt from his balance
    2. The exercised `tapOft` tokens remain stuck forever in the USDO contract because they are never actually bridged
2. The most probable: `msg.sender` (LayerZero endpoint) does not have enough `amountSentLD` of USDO tokens to be burnt. In this case, an error will be thrown and the whole call will revert, leading to a DoS

## Proof of Concept

The following poc shows how the function will be DoS’ed due to the sender not having enough USDO to be burnt. In order to execute the Poc, perform the following steps:

1. Remove the `_checkWhitelistStatus(OFTMsgCodec.bytes32ToAddress(msg_.lzSendParams.sendParam.to));` line in `UsdoOptionReceiverModule.sol`'s `exerciseOptionsReceiver()` function (it is wrong and related to another vulnerability)
2. Paste the following code in `Tapioca-bar/test/Usdo.t.sol`:
    
    ```solidity
    // Usdo.t.sol
    
    function testVuln_exercise_option() public {
            uint256 erc20Amount_ = 1 ether;
    
            //setup
            {
                deal(address(aUsdo), address(this), erc20Amount_);
    
                // @dev send TAP to tOB
                deal(address(tapOFT), address(tOB), erc20Amount_);
    
                // @dev set `paymentTokenAmount` on `tOB`
                tOB.setPaymentTokenAmount(erc20Amount_);
            }
     
            //useful in case of withdraw after borrow
            LZSendParam memory withdrawLzSendParam_;
            MessagingFee memory withdrawMsgFee_; // Will be used as value for the composed msg
    
            {
                // @dev `withdrawMsgFee_` is to be airdropped on dst to pay for the send to source operation (B->A).
                PrepareLzCallReturn memory prepareLzCallReturn1_ = usdoHelper.prepareLzCall( // B->A data
                    IUsdo(address(bUsdo)),
                    PrepareLzCallData({
                        dstEid: aEid,
                        recipient: OFTMsgCodec.addressToBytes32(address(this)),
                        amountToSendLD: erc20Amount_,
                        minAmountToCreditLD: erc20Amount_,
                        msgType: SEND,
                        composeMsgData: ComposeMsgData({
                            index: 0,
                            gas: 0,
                            value: 0,
                            data: bytes(""),
                            prevData: bytes(""),
                            prevOptionsData: bytes("")
                        }),
                        lzReceiveGas: 500_000,
                        lzReceiveValue: 0
                    })
                );
                withdrawLzSendParam_ = prepareLzCallReturn1_.lzSendParam;
                withdrawMsgFee_ = prepareLzCallReturn1_.msgFee;
            }
    
            /**
             * Actions
             */
            uint256 tokenAmountSD = usdoHelper.toSD(erc20Amount_, aUsdo.decimalConversionRate());
    
            //approve magnetar
            ExerciseOptionsMsg memory exerciseMsg = ExerciseOptionsMsg({
                optionsData: IExerciseOptionsData({
                    from: address(this),
                    target: address(tOB), 
                    paymentTokenAmount: tokenAmountSD,
                    oTAPTokenID: 0, // @dev ignored in TapiocaOptionsBrokerMock
                    tapAmount: tokenAmountSD
                }),
                withdrawOnOtherChain: true,
                lzSendParams: LZSendParam({
                    sendParam: SendParam({
                        dstEid: 0,
                        to: "0x",
                        amountLD: erc20Amount_,
                        minAmountLD: erc20Amount_,
                        extraOptions: "0x",
                        composeMsg: "0x",
                        oftCmd: "0x"
                    }),
                    fee: MessagingFee({nativeFee: 0, lzTokenFee: 0}),
                    extraOptions: "0x",
                    refundAddress: address(this)
                }),
                composeMsg: "0x"
            });
            bytes memory sendMsg_ = usdoHelper.buildExerciseOptionMsg(exerciseMsg);
    
            PrepareLzCallReturn memory prepareLzCallReturn2_ = usdoHelper.prepareLzCall(
                IUsdo(address(aUsdo)),
                PrepareLzCallData({
                    dstEid: bEid,
                    recipient: OFTMsgCodec.addressToBytes32(address(this)),
                    amountToSendLD: erc20Amount_,
                    minAmountToCreditLD: erc20Amount_,
                    msgType: PT_TAP_EXERCISE,
                    composeMsgData: ComposeMsgData({
                        index: 0,
                        gas: 500_000,
                        value: uint128(withdrawMsgFee_.nativeFee),
                        data: sendMsg_,
                        prevData: bytes(""),
                        prevOptionsData: bytes("")
                    }),
                    lzReceiveGas: 500_000,
                    lzReceiveValue: 0
                })
            );
            bytes memory composeMsg_ = prepareLzCallReturn2_.composeMsg;
            bytes memory oftMsgOptions_ = prepareLzCallReturn2_.oftMsgOptions;
            MessagingFee memory msgFee_ = prepareLzCallReturn2_.msgFee;
            LZSendParam memory lzSendParam_ = prepareLzCallReturn2_.lzSendParam;
    
            (MessagingReceipt memory msgReceipt_,) = aUsdo.sendPacket{value: msgFee_.nativeFee}(lzSendParam_, composeMsg_);
    
            {
                verifyPackets(uint32(bEid), address(bUsdo));
    
                vm.expectRevert("ERC20: burn amount exceeds balance");
                this.lzCompose(
                    bEid,
                    address(bUsdo),
                    oftMsgOptions_,
                    msgReceipt_.guid,
                    address(bUsdo),
                    abi.encodePacked(
                        OFTMsgCodec.addressToBytes32(address(this)), composeMsg_
                    )
            ); 
    
            }
    
        }
    ```
    
3. Run the poc with the following command, inside the Tapioca-bar repo: `forge test --mt testVuln_exercise_option`

We can see how the "ERC20: burn amount exceeds balance"  error is thrown due to the issue mentioned in the report.

## Impact

High. As demonstrated, two critical outcomes might affect the user:

1.  `tapOft` funds will remain stuck forever in the USDO contract and USDO will be incorrectly burnt from `msg.sender`
2. The core functionality of exercising and bridging options always reverts and effectively causes a DoS.

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/modules/UsdoOptionReceiverModule.sol#L120-L121

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/modules/UsdoOptionReceiverModule.sol#L149-L159

## Tool used

Manual Review, foundry

## Recommendation

If users decide to bridge their exercised tapOft, the sendPacket() function incorporated in the tapOft contract should be used instead of UsdoOptionReceiverModule’s internal _sendPacket() function, so that the actual bridged asset is the tapOft and not the USDO.
