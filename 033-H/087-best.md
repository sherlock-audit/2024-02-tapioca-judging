Zealous Pineapple Duck

high

# TOFTOptionsReceiverModule miss cross-chain transformation for deposit and lock amounts

## Summary

Cross-chain token decimals transformation is applied partially in TOFTOptionsReceiverModule's `lockAndParticipateReceiver()` and `mintLendXChainSGLXChainLockAndParticipateReceiver()`.

## Vulnerability Detail

Currently only first level amounts are being transformed in cross-chain TOFTOptionsReceiverModule, while the nested deposit and lock amounts involved aren't.

Whenever the decimals are different for underlying tokens across chains the absence of transformation will lead to magnitudes sized misrepresentation of user operations, which can result in core functionality unavailability (operations can constantly revert or become a noops due to running them with outsized or dust sized parameters) and loss of user funds (when an operation was successfully run, but with severely misrepresented parameters).

## Impact

Probability can be estimated as medium due to prerequisite of having asset decimals difference between transacting chains, while the operation misrepresentation and possible fund loss impact described itself has high severity.

Likelihood: Medium + Impact: High = Severity: High.

## Code Snippet

Only `mintAmount` is being transformed in `mintLendXChainSGLXChainLockAndParticipateReceiver()`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L72-L82

```solidity
    function mintLendXChainSGLXChainLockAndParticipateReceiver(bytes memory _data) public payable {
        // Decode received message.
        CrossChainMintFromBBAndLendOnSGLData memory msg_ =
            TOFTMsgCodec.decodeMintLendXChainSGLXChainLockAndParticipateMsg(_data);

        _checkWhitelistStatus(msg_.bigBang);
        _checkWhitelistStatus(msg_.magnetar);

        if (msg_.mintData.mintAmount > 0) {
            msg_.mintData.mintAmount = _toLD(msg_.mintData.mintAmount.toUint64());
        }
```

But collateral deposit amount from `CrossChainMintFromBBAndLendOnSGLData.mintData.collateralDepositData` there isn't:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/gitmodule/tapioca-periph/contracts/interfaces/periph/IMagnetar.sol#L104-L111

```solidity
struct CrossChainMintFromBBAndLendOnSGLData {
    address user;
    address bigBang;
    address magnetar;
    address marketHelper;
>>  IMintData mintData;
    LendOrLockSendParams lendSendParams;
}
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/gitmodule/tapioca-periph/contracts/interfaces/oft/IUsdo.sol#L136-L140

```solidity
struct IMintData {
    bool mint;
    uint256 mintAmount;
>>  IDepositData collateralDepositData;
}
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/gitmodule/tapioca-periph/contracts/interfaces/common/ICommonData.sol#L22-L25

```solidity
struct IDepositData {
    bool deposit;
>>  uint256 amount;
}
```

Similarly option lock's `amount` and `fraction` from `LockAndParticipateData` in `lockAndParticipateReceiver()`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L106-L121

```solidity
    function lockAndParticipateReceiver(bytes memory _data) public payable {
        // Decode receive message
        LockAndParticipateData memory msg_ = TOFTMsgCodec.decodeLockAndParticipateMsg(_data);

        _checkWhitelistStatus(msg_.magnetar);
        _checkWhitelistStatus(msg_.singularity);
        if (msg_.lockData.lock) {
            _checkWhitelistStatus(msg_.lockData.target);
        }
        if (msg_.participateData.participate) {
            _checkWhitelistStatus(msg_.participateData.target);
        }

        if (msg_.fraction > 0) {
            msg_.fraction = _toLD(msg_.fraction.toUint64());
        }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/gitmodule/tapioca-periph/contracts/interfaces/periph/IMagnetar.sol#L135-L142

```solidity
struct LockAndParticipateData {
    address user;
    address singularity;
    address magnetar;
    uint256 fraction;
>>  IOptionsLockData lockData;
    IOptionsParticipateData participateData;
}
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/gitmodule/tapioca-periph/contracts/interfaces/tap-token/ITapiocaOptionLiquidityProvision.sol#L30-L36

```solidity
struct IOptionsLockData {
    bool lock;
    address target;
    uint128 lockDuration;
>>  uint128 amount;
>>  uint256 fraction;
}
```

## Tool used

Manual Review

## Recommendation

Consider adding these local decimals transformations, e.g.:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L80-L82

```diff
        if (msg_.mintData.mintAmount > 0) {
            msg_.mintData.mintAmount = _toLD(msg_.mintData.mintAmount.toUint64());
        }
+       if (msg_.mintData.collateralDepositData.amount > 0) {
+           msg_.mintData.collateralDepositData.amount = _toLD(msg_.mintData.collateralDepositData.amount.toUint64());
+       }
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTOptionsReceiverModule.sol#L112-L114

```diff
        if (msg_.lockData.lock) {
            _checkWhitelistStatus(msg_.lockData.target);
+           if (msg_.lockData.amount > 0) msg_.lockData.amount = _toLD(msg_.lockData.amount.toUint64());
+           if (msg_.lockData.fraction > 0) msg_.lockData.fraction = _toLD(msg_.lockData.fraction.toUint64());
        }
```
