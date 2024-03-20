Skinny Wool Mallard

medium

# Secondary Big Bang market rates can be manipulated due to not triggering penrose.reAccrueBigBangMarkets(); when leveraging

## Summary

Secondary market rates can still be manipulated via leverage executors because `penrose.reAccrueBigBangMarkets()` is never called in the leverage module.

## Vulnerability Detail

The attack described in [Tapioca’s C4 audit 1561 issue](https://github.com/code-423n4/2023-07-tapioca-findings/issues/1561) and also described in Spearbit’s audit 5.2.16 issue is still possible utilizing the leverage modules.

As a summary, these attacks described a way to manipulate interest rates. As stated in [Tapioca’s documentation](https://docs.tapioca.xyz/tapioca/core-technologies/big-bang#variable-interest-rate-collateral-debt-ratio), the interest rate for non-ETH markets is computed considering the current debt in ETH markets. Rate manipulation could be performed by an attacker following these steps:

1. Borrow a huge amount in the ETH market. This step did not accrue the other markets.
2. Accrue other non-ETH markets. It is important to be aware of the fact that non-ETH markets base their interest calculations considering the total debt in the ETH market. After step 1, the attacker triggers an accrual on non-ETH markets which will fetch the data from the greatly increased  borrow amount in the ETH market, making the non-ETH market see a huge amount of debt, thus affecting and manipulating the computation of its interest rate.

The fix introduced in the C4 and Spearbit audits incorporated a new function in the Penrose contract to mitigate this issue. If the caller is the `bigBangEthMarket`, then the internal `_reAccrueMarkets()` function will be called, and market’s interest rates will be accrued prior to performing any kind of borrow. Following this fix, an attacker can no longer perform step 2 of accruing the markets with a manipulated rate because accrual on secondary markets has already been triggered.

```solidity
// Penrose.sol

function reAccrueBigBangMarkets() external notPaused {
        if (msg.sender == bigBangEthMarket) {
            _reAccrueMarkets(false);
        } 
    }
    
  function _reAccrueMarkets(bool includeMainMarket) private {
      uint256 len = allBigBangMarkets.length;
      address[] memory markets = allBigBangMarkets;
      for (uint256 i; i < len; i++) {
          address market = markets[i];
          if (isMarketRegistered[market]) {
              if (includeMainMarket || market != bigBangEthMarket) {
                  IBigBang(market).accrue();
              }
          }
      }

      emit ReaccruedMarkets(includeMainMarket);
  }
```

Although this fix is effective, the attack is still possible via Big Bang’s leverage modules. Leveraging is a different way of borrowing that still affects a market’s total debt. As we can see, the `buyCollateral()` function still performs a `_borrow()`, thus incrementing a market’s debt:

```solidity
// BBLeverage.sol

function buyCollateral(address from, uint256 borrowAmount, uint256 supplyAmount, bytes calldata data) 
        external
        optionNotPaused(PauseType.LeverageBuy)
        solvent(from, false)
        notSelf(from)  
        returns (uint256 amountOut) 
    { 
        ...

        
        {
            (, uint256 borrowShare) = _borrow( 
                calldata_.from,    
                address(this), 
                calldata_.borrowAmount,
                _computeVariableOpeningFee(calldata_.borrowAmount)
            );  
            (memoryData.borrowShareToAmount,) =
                yieldBox.withdraw(assetId, address(this), address(leverageExecutor), 0, borrowShare);
        }
        
        ...
       }
```

Because Penrose’s `reAccrueBigBangMarkets()` function is not called when leveraging, the attack described in the C4 and Spearbit audits is still possible by utilizing leverage to increase the ETH market’s total debt, and then accruing non-ETH markets so that rates are manipulated.

## Impact

Medium. A previously found issue is still present in the codebase which allows secondary Big Bang markets interest rates to be manipulated, allowing the attacker to perform profitable strategies and potentially affecting users.

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L53

## Tool used

Manual Review

## Recommendation

It is recommended to trigger Penrose’s reAccrueBigBangMarkets() function when interacting with Big Bang’s leverage modules, so that the issue can be fully mitigated.
