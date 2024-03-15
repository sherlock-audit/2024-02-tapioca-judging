Future Azure Locust

high

# Unverified `_srcChainSender` parameter allows to impersonate the sender

## Summary

The `_toeComposeReceiver` function accepts the `_srcChainSender` parameter that represents the sender of cross-chain message (via LayerZero's OFT) on the source chain. The function executes modules depending on the `_msgType` parameter and some of them do not accept the `_srcChainSender` parameter. Lack of verification for `_srcChainSender` means that the attacker can execute those modules on behalf of different users.

## Vulnerability Detail

The `_toeComposeReceiver` function is called by the LayerZero endpoint (indirectly) when there is a compose message to be executed. It gets three parameters:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/modules/UsdoReceiver.sol#L63-L100

The first patameter (`_msgType`) represents the type of message that should be executed on the destination chain. The second (`_srcChainSender`) is the sender of the message on the source chain and last one (`_toeComposeMsg`) contains the parameters for the executed operation.

In case of `MSG_TAP_EXERCISE` the `_srcChainSender` parameter is forwarded to the `UsdoOptionReceiver` module:
https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/modules/UsdoReceiver.sol#L68-L75

In case of other types (`MSG_MARKET_REMOVE_ASSET`, `MSG_YB_SEND_SGL_LEND_OR_REPAY` and `MSG_DEPOSIT_LEND_AND_SEND_FOR_LOCK`) the `_srcChainSender` parameter is not forwarder and the attacker fully control the contents of `_toeComposeMsg`.

**Let's take the `MSG_MARKET_REMOVE_ASSET` message as an example.**

1. The `removeAssetReceiver` function from `UsdoMarketReceiverModule` is executed with `_toeComposeMsg` parameter.

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/modules/UsdoMarketReceiverModule.sol#L210-L241

2. The `_toeComposeMsg` bytes (called `_data` in this function) are decoded and some values are extracted. The most important are: 
* `msg_.externalData.magnetar` on which the `burst` function is later called with specific magnetar calls (it is legitimate and whitelisted magnetar),
* `msg_.user` on whose behalf the further operation is called,
* `msg_.externalData` which is forwarder to further call,
* `msg_.removeAndRepayData` which is forwarder to further call.

Those parameters are used to prepare a call to `exitPositionAndRemoveCollateral` function from `OptionModule` module (defined in action's `id` param). 

3. Next, the `burst` function from magnetar contract is called and it executes the specific module depending on the `_action.id`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/gitmodule/tapioca-periph/contracts/Magnetar/Magnetar.sol#L138-L141

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/gitmodule/tapioca-periph/contracts/Magnetar/modules/MagnetarOptionModule.sol#L58-L210

4. The modules validates the sender passing the user address as the parameter.

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/gitmodule/tapioca-periph/contracts/Magnetar/modules/MagnetarOptionModule.sol#L60

5. The `_checkSender` function does not revert if the user is the sender or the sender is whitelisted.

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/gitmodule/tapioca-periph/contracts/Magnetar/MagnetarStorage.sol#L93-L97

6. In this case the sender is USDO contract which is whitelisted. This allows to continue operations from `exitPositionAndRemoveCollateral` function on behalf of the user (who is the victim).

**Note:** *This is only one of possible attack scenarios that exploits lack of `_srcChainSender` parameter validation.*

## Impact

HIGH - The attacker can execute functions from `UsdoMarketReceiverModule` module on behalf of any user. 

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/modules/UsdoReceiver.sol#L79

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/modules/UsdoReceiver.sol#L85

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/modules/UsdoReceiver.sol#L92

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/BaseTOFTReceiver.sol#L70

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/BaseTOFTReceiver.sol#L76

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/BaseTOFTReceiver.sol#L98

## Tool used

Manual Review

## Recommendation

Validate whether the user whose assets are being managed is the same address as the `_srcChainSender` parameter. 