Skinny Wool Mallard

medium

# Big Bang debt rate is computed using an outdated total debt from the ETH market

## Summary

Not updating the ETH market's interest will make secondary market interest rates computations be always outdated.

## Vulnerability Detail

Every time a critical interaction is performed in a Big Bang market, an interest accrual must be performed so that the protocol interactions operate using the most updated interest rate values.

Tapioca’s Big Bang markets incorporate a variable interest rate model following the concept of [Collateral Debt Ratio](https://docs.tapioca.xyz/tapioca/core-technologies/big-bang#variable-interest-rate-collateral-debt-ratio), where all collateral assets will feature a **debt ratio** against ETH (the ETH collateral market is the only market that will feature a **fixed** interest rate).This means that debt computed for non-ETH markets will be based and computed considering the current ETH market debt.

In order to accrue interest, every relevant market interaction will trigger the internal `_accrue()` function so that interest can be accrued.

As we can see in the following code snippet, `_accrue()` will call  `getDebtRate()` in order to obtain the current rate so that the corresponding interest value can be computed:

```solidity
// BBCommon.sol

function _accrue() internal override {    
        IBigBang.AccrueInfo memory _accrueInfo = accrueInfo;
        // Number of seconds since accrue was called
        uint256 elapsedTime = block.timestamp - _accrueInfo.lastAccrued; 
        if (elapsedTime == 0) {
            return; 
        }  
        //update debt rate
        uint256 annumDebtRate = getDebtRate(); 
        _accrueInfo.debtRate = (annumDebtRate / 31557600).toUint64(); //per second; account for leap years
        
        
        ...
    }
```

`getDebtRate()` is a crucial function that will return the debt considering the current state of the markets. If the current market is the ETH market (`isMainMarket` is `true`), then a fixed debt rate will be applied by calling `penrose.bigBangEthDebtRate()`. Otherwise, debt is computed by obtaining the ETH market’s total debt, and then performing other computations based on the queried ETH market’s debt:

```solidity
// BBCommon.sol
function getDebtRate() public view returns (uint256) {
        if (isMainMarket) return penrose.bigBangEthDebtRate(); // default 0.5% 
        if (totalBorrow.elastic == 0) return minDebtRate;
  
        uint256 _ethMarketTotalDebt = IBigBang(penrose.bigBangEthMarket()).getTotalDebt(); 
        
        ...
    } 
```

The problem with this approach is that `IBigBang(penrose.bigBangEthMarket()).getTotalDebt();` will always return an outdated total debt value, making the current market’s interest computations be wrong. 

When queried, `getTotalDebt()` will simply return the ETH’s market `totalBorrow.elastic`  value:

```solidity
// BBCommon.sol

function getTotalDebt() external view returns (uint256) {
        return totalBorrow.elastic; 
    }
```

`totalBorrow.elastic` is updated when the `accrue()` function is triggered. Because an accrual is not performed for the ETH market when a regular market is performing an accrual of interest, the `totalBorrow.elastic` value queried from the ETH market will be outdated, leading to an improper calculation of the current market’s interest rates.

## Impact

Medium. Interest rates will always be outdated and wrongly computed in non-ETH markets because the ETH market is not properly updated, leading to the market rates not functioning in an optimal way and potentially affecting USDO’s peg.

## Code Snippet

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBCommon.sol#L49

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBCommon.sol#L40-L42

## Tool used

Manual Review

## Recommendation

Force an ETH market interest accrual if the current market is non-ETH so that the interest can be computed in the proper way:

```diff
// BBCommon.sol

function getDebtRate() public view returns (uint256) {
        if (isMainMarket) return penrose.bigBangEthDebtRate(); // default 0.5% 
        if (totalBorrow.elastic == 0) return minDebtRate;
			  
+	 IBigBang(penrose.bigBangEthMarket()).accrue();
        uint256 _ethMarketTotalDebt = IBigBang(penrose.bigBangEthMarket()).getTotalDebt(); 
        uint256 _currentDebt = totalBorrow.elastic;   
        uint256 _maxDebtPoint = (_ethMarketTotalDebt * debtRateAgainstEthMarket) / 1e18;
 
        if (_currentDebt >= _maxDebtPoint) return maxDebtRate;

        uint256 debtPercentage = (_currentDebt * DEBT_PRECISION) / _maxDebtPoint;
        uint256 debt = ((maxDebtRate - minDebtRate) * debtPercentage) / DEBT_PRECISION + minDebtRate;

        if (debt > maxDebtRate) return maxDebtRate;

        return debt;
    } 
```
