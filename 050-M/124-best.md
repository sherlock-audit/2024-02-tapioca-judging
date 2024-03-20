Skinny Wool Mallard

high

# USDO’s MSG_TAP_EXERCISE compose messages where exercised options must be withdrawn to another chain will always fail due to wrongly requiring sendParam's to address to be whitelisted in the Cluster

## Summary

Wrongly checking for the sendParam's `to` address to be whitelisted when bridging exercised options will make such calls always fail.

## Vulnerability Detail

One of the compose messages allowed in USDO is `MSG_TAP_EXERCISE`. This type of message will trigger `UsdoOptionReceiverModule`'s `exerciseOptionsReceiver()` function, which allows users to exercise their options and obtain the corresponding exercised tapOFTs.

Users can choose to obtain their tapOFTs in the chain where `exerciseOptionsReceiver()` is being executed, or they can choose to send a message to a destination chain of their choice. If users decide to bridge the exercised option, the `lzSendParams` fields contained in the `ExerciseOptionsMsg` struct decoded from the `_data` passed as parameter in `exerciseOptionsReceiver()` should be filled with the corresponding data to perform the cross-chain call.

The problem is that  the `exerciseOptionsReceiver()` performs an unnecessary validation that requires the `to` parameter inside the `lzSendParams` to be whitelisted in the protocol’s cluster:

```solidity
// UsdoOptionReceiverModule.sol

function exerciseOptionsReceiver(address srcChainSender, bytes memory _data) public payable {
        // Decode received message.
        ExerciseOptionsMsg memory msg_ = UsdoMsgCodec.decodeExerciseOptionsMsg(_data);
  
        _checkWhitelistStatus(msg_.optionsData.target);
        _checkWhitelistStatus(OFTMsgCodec.bytes32ToAddress(msg_.lzSendParams.sendParam.to)); // <---- This validation is wrong 
        ...
        
        
   }
```

`msg_.lzSendParams.sendParam.to` corresponds to the address that will obtain the tokens in the destination chain after bridging the exercised option, which can and should actually be any address that the user exercising the option decides, so this address shouldn’t be required to be whitelisted in the protocol’s Cluster (given that the Cluster only whitelists certain protocol-related addresses such as contracts or special addresses).

Because of this, transactions where users try to bridge the exercised options will always fail because the  `msg_.lzSendParams.sendParam.to` address specified by users will never be whitelisted in the Cluster.

## Impact

High. The functionality of exercising options and bridging them in the same transaction is one of the wide range of core functionalities that should be completely functional in Tapioca. However, this functionality will always fail due to the mentioned issue, forcing users to only be able to exercise options in the same chain.

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/modules/UsdoOptionReceiverModule.sol#L72

## Tool used

Manual Review

## Recommendation

Remove the whitelist check against the `msg_.lzSendParams.sendParam.to` param in`exerciseOptionsReceiver()`:

```diff
// UsdoOptionReceiverModule.sol

function exerciseOptionsReceiver(address srcChainSender, bytes memory _data) public payable {
        // Decode received message.
        ExerciseOptionsMsg memory msg_ = UsdoMsgCodec.decodeExerciseOptionsMsg(_data);
  
        _checkWhitelistStatus(msg_.optionsData.target);
-        _checkWhitelistStatus(OFTMsgCodec.bytes32ToAddress(msg_.lzSendParams.sendParam.to)); 
        ...
        
        
   }
```
