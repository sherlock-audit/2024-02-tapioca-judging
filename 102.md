Vast Bronze Salamander

high

# exerciseOptionsReceiver() Lack of Ownership Check for oTAP, Allowing Anyone to Use oTAPTokenID

## Summary
In `UsdoOptionReceiverModule.exerciseOptionsReceiver()`:
For this method to execute successfully, the `owner` of the `oTAPTokenID` needs to approve it to `address(usdo)`.
Once approved, anyone can front-run execute `exerciseOptionsReceiver()` and utilize this authorization.

## Vulnerability Detail
In `USDO.lzCompose()`, it is possible to specify `_msgType == MSG_TAP_EXERCISE` to execute `USDO.exerciseOptionsReceiver()` across chains.

```solidity
    function exerciseOptionsReceiver(address srcChainSender, bytes memory _data) public payable {
...
            ITapiocaOptionBroker(_options.target).exerciseOption(
@>              _options.oTAPTokenID,
                address(this), //payment token
                _options.tapAmount
            );
            _approve(address(this), address(pearlmit), 0);
            uint256 bAfter = balanceOf(address(this));

            // Refund if less was used.
            if (bBefore > bAfter) {
                uint256 diff = bBefore - bAfter;
                if (diff < _options.paymentTokenAmount) {
                    IERC20(address(this)).safeTransfer(_options.from, _options.paymentTokenAmount - diff);
                }
            }
...
```
For this method to succeed, USDO must first obtain approve for the `oTAPTokenID`.

Example: The owner of `oTAPTokenID` is Alice.
1. alice in A chain execute lzSend(dstEid = B)  with
    - composeMsg = [oTAP.permit(usdo,oTAPTokenID,v,r,s) 2.exerciseOptionsReceiver(oTAPTokenID,_options.from=alice) 3. oTAP.revokePermit(oTAPTokenID)]
2. in chain B USDO.lzCompose() will 
   - execute oTAP.permit(usdo,oTAPTokenID)
   - exerciseOptionsReceiver(srcChainSender=alice,_options.from=alice,oTAPTokenID ) 
   - oTAP.revokePermit(oTAPTokenID)
   
The signature of `oTAP.permit` is public, allowing anyone to use it. 
>Note: if alice call approve(oTAPTokenID,usdo) in chain B  without signature, but The same result

This opens up the possibility for malicious users to front-run use this signature. 
Let's consider an example with Bob:
1. Bob in Chain A uses Alice's signature (v, r, s):
    - `composeMsg = [oTAP.permit(usdo, oTAPTokenID, v, r, s), exerciseOptionsReceiver(oTAPTokenID, _options.from=bob)]`-----> (Note: `_options.from` should be set to Bob.)
2. In Chain B, when executing `USDO.lzCompose(dstEid = B)`, the following actions occur:
    - Execute `oTAP.permit(usdo, oTAPTokenID)`
    - Execute `exerciseOptionsReceiver(srcChainSender=bob, _options.from=bob, oTAPTokenID)`

As a result, Bob gains unconditional access to this `oTAPTokenID`.

It is advisable to check the ownership of `oTAPTokenID` is `_options.from` before executing `ITapiocaOptionBroker(_options.target).exerciseOption()`.

## Impact

The `exerciseOptionsReceiver()` function lacks ownership checks for `oTAP`, allowing anyone to use `oTAPTokenID`.

## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/modules/UsdoOptionReceiverModule.sol#L67
## Tool used

Manual Review

## Recommendation

add check `_options.from` is owner or be approved

```diff
    function exerciseOptionsReceiver(address srcChainSender, bytes memory _data) public payable {

...
            uint256 bBefore = balanceOf(address(this));
+           address oTap = ITapiocaOptionBroker(_options.target).oTAP();
+           address oTapOwner = IERC721(oTap).ownerOf(_options.oTAPTokenID);
+           require(oTapOwner == _options.from
+                         || IERC721(oTap).isApprovedForAll(oTapOwner,_options.from)
+                         || IERC721(oTap).getApproved(_options.oTAPTokenID) == _options.from
+                        ,"invalid");
            ITapiocaOptionBroker(_options.target).exerciseOption(
                _options.oTAPTokenID,
                address(this), //payment token
                _options.tapAmount
            );
            _approve(address(this), address(pearlmit), 0);
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
