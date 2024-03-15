Vast Bronze Salamander

medium

# Multiple contracts cannot be paused

## Summary
For safety, tapioca has added `whenNotPaused` restrictions to multiple contracts
But there is no method provided to modify the `_paused` state
If a security event occurs, it cannot be paused at all

## Vulnerability Detail
Take `mTOFT.sol` as an example, multiple methods are `whenNotPaused`
```solidity
    function executeModule(ITOFT.Module _module, bytes memory _data, bool _forwardRevert)
        external
        payable
@>      whenNotPaused
        returns (bytes memory returnData)
    {
...
    function sendPacket(LZSendParam calldata _lzSendParam, bytes calldata _composeMsg)
        public
        payable
@>      whenNotPaused
        returns (MessagingReceipt memory msgReceipt, OFTReceipt memory oftReceipt)
    {

```

But the contract does not provide a `public` method to modify `_paused`
Note: `Pausable.sol` does not have a `public` method to modify `_paused`

In reality, there have been multiple reports of security incidents where the protocol side wants to pause to prevent losses, but cannot pause, strongly recommend adding

Note: The following contracts cannot be paused
- mTOFT
- TOFT
- Usdo
- AssetToSGLPLeverageExecutor

## Impact

Due to the inability to modify `_paused`, it poses a security risk

## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/mTOFT.sol#L50
## Tool used

Manual Review

## Recommendation
```diff
+    function pause() external onlyOwner{
+        _pause();
+    }

+    function unpause() external onlyOwner{
+        _unpause();
+    }
```
