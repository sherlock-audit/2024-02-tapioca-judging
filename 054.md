Vast Bronze Salamander

medium

# ERC4494.sol is not compatible with ERC-4494

## Summary

`ERC721Permit` uses the owner's nonce, not the tokenId's nonce defined by EIP. 
And the nonce is not modified after the tokenId is transferred, which poses a certain security risk.

## Vulnerability Detail
In the definition of `eip-4494`, it is the nonce of tokenId
>https://eips.ethereum.org/EIPS/eip-4494
nonces[tokenId] is equal to nonce

The implementation of Uniswap3 is also based on EIP-4494
https://github.com/Uniswap/v3-periphery/blob/697c2474757ea89fec12a4e6db16a574fe259610/contracts/base/ERC721Permit.sol#L16
```solidity
abstract contract ERC721Permit is BlockTimestamp, ERC721, IERC721Permit {
    /// @dev Gets the current nonce for a token ID and then increments it, returning the original value
    function _getAndIncrementNonce(uint256 tokenId) internal virtual returns (uint256);

```

But currently, `ERC721Permit` is implemented based on the owner
```solidity
abstract contract ERC721Permit is ERC721, EIP712 {
    using Counters for Counters.Counter;

    mapping(address => Counters.Counter) private _nonces;
...
    function _useNonce(address owner) internal virtual returns (uint256 current) {
        Counters.Counter storage nonce = _nonces[owner];
        current = nonce.current();
        nonce.increment();
    }
```

Also, for security reasons, it is recommended to increase the nonce after transfer
>eip-4494
>the nonce of a particular tokenId (nonces[tokenId]) MUST be incremented upon any transfer of the tokenId

## Impact

It is not compatible with eip, standard clients/wallets cannot normally permit

## Code Snippet
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/util/ERC4494.sol#L49

## Tool used

Manual Review

## Recommendation
1. use _nonces[tokenId]
2. add _afterTokenTransfer()
```diff
abstract contract ERC721Permit is ERC721, EIP712 {
    using Counters for Counters.Counter;

-   mapping(address => Counters.Counter) private _nonces;
+   mapping(uint256 => Counters.Counter) private _nonces;

    function permit(address spender, uint256 tokenId, uint256 deadline, uint8 v, bytes32 r, bytes32 s) public virtual {
        require(block.timestamp <= deadline, "ERC721Permit: expired deadline");

        address owner = ownerOf(tokenId);
-       bytes32 structHash = keccak256(abi.encode(_PERMIT_TYPEHASH, spender, tokenId, _useNonce(owner), deadline));
+       bytes32 structHash = keccak256(abi.encode(_PERMIT_TYPEHASH, spender, tokenId, _useNonce(tokenId), deadline));
...
     }
-    function nonces(address owner) public view virtual returns (uint256) {
-        return _nonces[owner].current();
-    }

+    function nonces(uint256 _tokenId) public view virtual returns (uint256) {
+        return _nonces[_tokenId].current();
+    }

-    function _useNonce(address owner) internal virtual returns (uint256 current) {
-        Counters.Counter storage nonce = _nonces[owner];
-        current = nonce.current();
-        nonce.increment();
-    }

+    function _useNonce(uint256 _tokenId) internal virtual returns (uint256 current) {
+        Counters.Counter storage nonce = _nonces[_tokenId];
+        current = nonce.current();
+        nonce.increment();
+    }

+    function _afterTokenTransfer(
+         address from,
+         address to,
+         uint256 firstTokenId,
+         uint256 batchSize
+     ) internal virtual override {
+         _useNonce(firstTokenId);
+         super._afterTokenTransfer(from, to, firstTokenId, batchSize);
+     } 
```