Gentle Sky Badger

medium

# decodeArrayOfYieldBoxPermitAssetMsg returns corrupted data

## Summary

False assumption made in the message length of YieldBoxApproveAssetMsg results in corrupted data.

## Vulnerability Detail

UsdoMsgCodec.decodeArrayOfYieldBoxPermitAssetMsg slices `_msg` into 190 bit slices under the assumption that each individual YieldBoxApproveAssetMsg contains 190 bits:

```solidity
uint256 msgCount_ = _msg.length / 190;

YieldBoxApproveAssetMsg[] memory approvalMsgs_ = new YieldBoxApproveAssetMsg[](msgCount_);

uint256 msgIndex_;
for (uint256 i; i < msgCount_;) {
    approvalMsgs_[i] = decodeYieldBoxApprovalAssetMsg(BytesLib.slice(_msg, msgIndex_, 190));
    unchecked {
        msgIndex_ += 190;
        ++i;
    }
}
```

The problem is that the packed data of buildYieldBoxPermitAssetMsg, as retrieved from buildYieldBoxPermitAssetMsg, actually contains 197 bits of data. The mistake the sponsor makes is in assuming that the permit bool at the very end requires only one bit. In reality, packed booleans require 8 bits.

We can test this quite simply by packing a uint256 value along with a boolean in chisel and checking the output.

```solidity
➜ abi.encodePacked(uint256(420), bool(true))
Type: dynamic bytes
├ Hex (Memory):
├─ Length ([0x00:0x20]): 0x0000000000000000000000000000000000000000000000000000000000000021
├─ Contents ([0x20:..]): 0x00000000000000000000000000000000000000000000000000000000000001a40100000000000000000000000000000000000000000000000000000000000000
```

We can see above that the length is 0x21 (33) bytes, with 0x20 (32) bytes for the uint256 as expected, but an additional **byte** which is 8 bits.

The result of this issue is that the returned array of approval messages is corrupted. The 0th element will be mostly correct but always report `permit` as false, whereas all other elements will be completely incorrect.

## Impact

Corrupted returned approval messages.

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/libraries/UsdoMsgCodec.sol#L134

## Tool used

Manual Review

## Recommendation

Use a message length of 197 instead of 190 to retrieve individual messages in this function.