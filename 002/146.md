Zealous Pineapple Duck

high

# mTOFT's fees cannot be paid on native wrapping

## Summary

mTOFT tries to pay the fees twice on native wrapping, so these operations will fail unless some extra donation is made.

## Vulnerability Detail

`mintFee` is being transferred twice, which will revert the most calls. The core functionality of native tokens wrapping is unavailable.

Zero fee isn't feasible for production, so native token wrapping will be unavailable in production.

## Impact

Wrapping of the native tokens into mTOFT is a base function of the protocol. Core contract functionality unavailability has high severity.

## Code Snippet

`_checkAndExtractFees()`, being called by `wrap()`, transfers `feeAmount = (_amount * mintFee) / 1e5` to the Vault:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/mTOFT.sol#L408-L423

```solidity
    function _checkAndExtractFees(uint256 _amount) private returns (uint256 feeAmount) {
        feeAmount = 0;

        // not on host chain; extract fee
        // fees are used to rebalance liquidity to host chain
        if (_getChainId() != hostEid && mintFee > 0) {
>>          feeAmount = (_amount * mintFee) / 1e5;
            if (feeAmount > 0) {
                if (erc20 == address(0)) {
>>                  vault.registerFees{value: feeAmount}(feeAmount);
                } else {
                    vault.registerFees(feeAmount);
                }
            }
        }
    }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/TOFTVault.sol#L78-L82

```solidity
    /// @notice register fees for mTOFT
>>  function registerFees(uint256 amount) external payable onlyOwner {
        if (msg.value > 0 && msg.value != amount) revert FeesAmountNotRight();
        _fees += amount;
    }
```

When it's native wrapping immediately thereafter `wrap()` calls `_wrapNative()`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/mTOFT.sol#L287-L306

```solidity
    function wrap(address _fromAddress, address _toAddress, uint256 _amount)
        external
        payable
        whenNotPaused
        nonReentrant
        returns (uint256 minted)
    {
        if (balancers[msg.sender]) revert mTOFT_BalancerNotAuthorized();
        if (!connectedChains[_getChainId()]) revert mTOFT_NotHost();
        if (mintCap > 0) {
            if (totalSupply() + _amount > mintCap) revert mTOFT_CapNotValid();
        }

>>      uint256 feeAmount = _checkAndExtractFees(_amount);
        if (erc20 == address(0)) {
>>          _wrapNative(_toAddress, _amount, feeAmount);
        } else {
            if (msg.value > 0) revert mTOFT_NotNative();
            _wrap(_fromAddress, _toAddress, _amount, feeAmount);
        }
```

Which tries to send the whole `_amount` to the Vault:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/BaseTOFT.sol#L78-L81

```solidity
    function _wrapNative(address _toAddress, uint256 _amount, uint256 _feeAmount) internal virtual {
>>      vault.depositNative{value: _amount}();
        _mint(_toAddress, _amount - _feeAmount);
    }
```

I.e. unless `_amount + feeAmount` is present on the contract balance, the operation will revert.

## Tool used

Manual Review

## Recommendation

Since `depositNative()` doesn't have any amount specific logic:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/TOFTVault.sol#L94-L98

```solidity
    /// @notice deposit native gas to vault
    function depositNative() external payable onlyOwner {
        if (!_isNative) revert NotValid();
        if (msg.value == 0) revert ZeroAmount();
    }
```

Consider reducing the amount attached to the deposit call, e.g.:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/BaseTOFT.sol#L78-L81

```diff
    function _wrapNative(address _toAddress, uint256 _amount, uint256 _feeAmount) internal virtual {
-       vault.depositNative{value: _amount}();
+       vault.depositNative{value: _amount - _feeAmount}();
        _mint(_toAddress, _amount - _feeAmount);
    }
```