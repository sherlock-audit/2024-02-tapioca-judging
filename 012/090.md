Zealous Pineapple Duck

high

# Malicious MarketHelper contract can be used in TOFTMarketReceiverModule's leverageUpReceiver and marketRemoveCollateralReceiver functions

## Summary

User-supplied `marketHelper` contract is called for building market's `execute()` call, but in `leverageUpReceiver()` and `marketRemoveCollateralReceiver()` it's not whitelisted.

## Vulnerability Detail

An attacker can craft any logic and provide it as `marketHelper`, placing arbitrary `modules` and `calls` for `market` `execute()`, not corresponding for `buyCollateral` or `removeCollateral` operations.

For example, `removeCollateral` operation can have both `msg_.withdrawParams.withdraw == true` and `to = msg_.user` instead of `to = msg_.removeParams.magnetar`, stealing the corresponding assets from `magnetar` balance (i.e. instead of forwarding the user assets received it will use assets from the own balance instead as user both received assets directly and called `withdraw` via `magnetar`).

## Impact

In the example above `magnetar`, being a helper contract itself, has to have assets on the balance to steal. But there might be different sequences of operations allowing other loss making manipulations. Placing to medium the cumulative probability of reaching the state when crafted `marketHelper` produced call sequence can trick the desired logic to gain a material benefit.

Likelihood: Medium + Impact: High = Severity: High.

## Code Snippet

`marketHelper` isn't checked to be whitelisted in `leverageUpReceiver()`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTMarketReceiverModule.sol#L74-L79

```solidity
    function leverageUpReceiver(bytes memory _data) public payable {
        /// @dev decode received message
        LeverageUpActionMsg memory msg_ = TOFTMsgCodec.decodeLeverageUpMsg(_data);

        /// @dev 'market'
        _checkWhitelistStatus(msg_.market);
```

It is used to craft call sequence for `market`, that can be arbitrary this way (which, even having `modules` fixed and sound, still can have a variety of unintended impacts similar to misplacing `to` as mentioned above):

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTMarketReceiverModule.sol#L88-L93

```solidity
        {
>>          (Module[] memory modules, bytes[] memory calls) = IMarketHelper(msg_.marketHelper).buyCollateral(
                msg_.user, msg_.borrowAmount, msg_.supplyAmount, msg_.executorData
            );
            IMarket(msg_.market).execute(modules, calls, true);
        }
```

And in `marketRemoveCollateralReceiver()`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTMarketReceiverModule.sol#L161-L165

```solidity
    function marketRemoveCollateralReceiver(bytes memory _data) public payable {
        /// @dev decode received message
        MarketRemoveCollateralMsg memory msg_ = TOFTMsgCodec.decodeMarketRemoveCollateralMsg(_data);

        _checkWhitelistStatus(msg_.removeParams.market);
```

Where it can, as an example, place `msg_.user` as `to`, still calling `withdraw` from `magnetar`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTMarketReceiverModule.sol#L172-L197

```solidity
        {
            uint256 share = IYieldBox(ybAddress).toShare(assetId, msg_.removeParams.amount, false);
            approve(msg_.removeParams.market, share);

>>          (Module[] memory modules, bytes[] memory calls) = IMarketHelper(msg_.removeParams.marketHelper)
                .removeCollateral(msg_.user, msg_.withdrawParams.withdraw ? msg_.removeParams.magnetar : msg_.user, share);
            IMarket(msg_.removeParams.market).execute(modules, calls, true);
        }

        {
>>          if (msg_.withdrawParams.withdraw) {
                _checkWhitelistStatus(msg_.removeParams.magnetar);

                bytes memory call =
                    abi.encodeWithSelector(MagnetarYieldBoxModule.withdrawToChain.selector, msg_.withdrawParams);
                MagnetarCall[] memory magnetarCall = new MagnetarCall[](1);
                magnetarCall[0] = MagnetarCall({
                    id: MagnetarAction.YieldBoxModule,
                    target: address(this),
                    value: msg.value,
                    allowFailure: false,
                    call: call
                });
                IMagnetar(payable(msg_.removeParams.magnetar)).burst{value: msg.value}(magnetarCall);
            }
        }
```

I.e. the assumption that the calls is constructed by regular white-listed `MarketHelper` to follow the operation logic is broken:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/MarketHelper.sol#L119-L128

```solidity
>>  function removeCollateral(address from, address to, uint256 share)
        external
        pure
        returns (Module[] memory modules, bytes[] memory calls)
    {
        modules = new Module[](1);
        calls = new bytes[](1);
        modules[0] = Module.Collateral;
>>      calls[0] = abi.encodeWithSelector(SGLCollateral.removeCollateral.selector, from, to, share);
    }
```

## Tool used

Manual Review

## Recommendation

Since the contract is known and already included to white lists in the system, consider checking it in `leverageUpReceiver()` and `marketRemoveCollateralReceiver()`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTMarketReceiverModule.sol#L74-L79

```diff
    function leverageUpReceiver(bytes memory _data) public payable {
        /// @dev decode received message
        LeverageUpActionMsg memory msg_ = TOFTMsgCodec.decodeLeverageUpMsg(_data);

        /// @dev 'market'
        _checkWhitelistStatus(msg_.market);
+       _checkWhitelistStatus(msg_.marketHelper);
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/modules/TOFTMarketReceiverModule.sol#L161-L165

```diff
    function marketRemoveCollateralReceiver(bytes memory _data) public payable {
        /// @dev decode received message
        MarketRemoveCollateralMsg memory msg_ = TOFTMsgCodec.decodeMarketRemoveCollateralMsg(_data);

        _checkWhitelistStatus(msg_.removeParams.market);
+       _checkWhitelistStatus(msg_.removeParams.marketHelper);
```