Zealous Pineapple Duck

medium

# TOFTOptionsReceiverModule's and UsdoOptionReceiverModule's exerciseOptionsReceiver can lose the option payment provided

## Summary

There is a valid case of zero `paymentAmount` in TapiocaOptionBroker's `exerciseOption()`. When this happens, `exerciseOptionsReceiver()` does not return any exercise funds remainder to the caller.

## Vulnerability Detail

Zero amount can happen due to rounding and is allowed in the logic. However, the reimbursement logic is conditioned on non-zero balance change (while it cannot be the case as the exercise reverts on all the errors, there is no possibility to just exit), so user will not be reimbursed in this case.

## Impact

The `_options.paymentTokenAmount` provided by the caller can be lost for them if `exerciseOption()` ended up requesting no payment due to rounding. These user provided funds can be immediately stolen by any back-running attacker, as attacker's `_options.paymentTokenAmount` can be less than what they need for exercise, i.e. currently anyone can freely use the funds from the contract balance to pay for their options' exercise as user provided funds aren't controlled to match with option strike payment ones.

The probability of such rounding can be estimated as low, while fund freezing impact is high.

Likelihood: Low + Impact: High = Severity: Medium.

## Code Snippet

TapiocaOptionBroker's `exerciseOption()` can request zero `paymentAmount` due to rounding in `_getDiscountedPaymentAmount()`:

https://github.com/Tapioca-DAO/tap-token/blob/main/contracts/options/TapiocaOptionBroker.sol#L592-L599

```solidity
        // Calculate payment amount
>>      paymentAmount = discountedOTCAmountInUSD / _paymentTokenValuation;

        if (_paymentTokenDecimals <= 18) {
>>          paymentAmount = paymentAmount / (10 ** (18 - _paymentTokenDecimals));
        } else {
            paymentAmount = paymentAmount * (10 ** (_paymentTokenDecimals - 18));
        }
```

Zero `discountedPaymentAmount` is allowed:

https://github.com/Tapioca-DAO/tap-token/blob/main/contracts/options/TapiocaOptionBroker.sol#L557-L574

```solidity
        // Calculate payment amount and initiate the transfers
>>      uint256 discountedPaymentAmount =
            _getDiscountedPaymentAmount(otcAmountInUSD, paymentTokenValuation, discount, _paymentToken.decimals());

        uint256 balBefore = _paymentToken.balanceOf(address(this));
        // IERC20(address(_paymentToken)).safeTransferFrom(msg.sender, address(this), discountedPaymentAmount);
        {
            bool isErr =
                pearlmit.transferFromERC20(msg.sender, address(this), address(_paymentToken), discountedPaymentAmount);
            if (isErr) revert TransferFailed();
        }
        uint256 balAfter = _paymentToken.balanceOf(address(this));
>>      if (balAfter - balBefore != discountedPaymentAmount) {
            revert TransferFailed();
        }

        tapOFT.extractTAP(msg.sender, tapAmount);
    }
```

In this case for `exerciseOptionsReceiver()` it is `bBefore == bAfter` and nothing will be refunded from `_options.paymentTokenAmount`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L174-L180

```solidity
            // Refund if less was used.
>>          if (bBefore > bAfter) {
                uint256 diff = bBefore - bAfter;
                if (diff < _options.paymentTokenAmount) {
                    IERC20(address(this)).safeTransfer(_options.from, _options.paymentTokenAmount - diff);
                }
            }
```

I.e. given `exerciseOption()` was run successfully the `bBefore == bAfter` state doesn't imply that no refund is needed.

In the same time `_options.paymentTokenAmount` is user provided and can be arbitrary large.

## Tool used

Manual Review

## Recommendation

Consider including zero amount case in TOFTOptionsReceiverModule's and UsdoOptionReceiverModule's `exerciseOptionsReceiver()` functions, e.g.:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L174-L180

```diff
            // Refund if less was used.
-           if (bBefore > bAfter) {
+           if (bBefore >= bAfter) {
                uint256 diff = bBefore - bAfter;
                if (diff < _options.paymentTokenAmount) {
                    IERC20(address(this)).safeTransfer(_options.from, _options.paymentTokenAmount - diff);
                }
            }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/modules/UsdoOptionReceiverModule.sol#L98-L104

```diff
            // Refund if less was used.
-           if (bBefore > bAfter) {
+           if (bBefore >= bAfter) {
                uint256 diff = bBefore - bAfter;
                if (diff < _options.paymentTokenAmount) {
                    IERC20(address(this)).safeTransfer(_options.from, _options.paymentTokenAmount - diff);
                }
            }
```
