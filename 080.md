Rural Amethyst Tapir

medium

# Airdopped tokens can be stolen by anyone

## Summary
`lzReceive` function inside `TOFT`, `mTOFT` and `USDO` contracts can be used to airdrop tokens to the contract. These tokens can be stolen by anyone.


## Vulnerability Detail

Cross-chain receiving logic inside `TOFT`, `mTOFT` and `USDO` is split between `lzReceive` and `lzCompose` functions.

`lzReceive` handles the crediting of tokens and invoking the `lzEndpoint.sendCompose` so the rest of the cross-chain flow can be executed inside the `lzCompose` function.

`lzCompose` in the context of the above-mentioned contracts handles many flows, some of which require the user to airdrop a certain amount of tokens so a specific action can be executed.
An example is withdrawing assets to another chain. 

If we check the LayerZero option types it is only possible to airdrop tokens as part of the `lzReceive` and not the `lzCompose` function call: https://docs.layerzero.network/contracts/options#lzcompose-option

This means that the airdropped tokens will be sitting in the `TOFT` contract for example in between the two calls.

This is a problem as some other user can directly use these gas tokens to pay for his LayerZero fees, effectively stealing the airdropped tokens.

## Impact
Any airdropped tokens as part of the `lzReceive` function call can be stolen by anyone.

## Code Snippet
- https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/TapiocaZ/contracts/tOFT/TOFT.sol#L168

## Tool used

Manual Review

## Recommendation
Disallow airdropping tokens as part of the `lzReceive` function call. This can be done by checking the `_extraOptions` passed to the `sendPacket` function. 