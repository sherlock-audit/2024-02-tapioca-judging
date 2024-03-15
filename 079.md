Zealous Pineapple Duck

medium

# Leverage borrowing with stale rate can atomically create bad debt with no prior positions and no investment

## Summary

Leverage buying, borrow and collateral removal can increase riskness of a position, but are allowed to be performed with a stale exchange rate within `rateValidDuration` both in BB ansd SGL. This can provide a way for creating bad debt whenever actual rate dropped more than `FEE_PRECISION - collateralizationRate` (`25%`).

Particularly, an attacker can atomically extract value from the protocol without having any prior positions via leverage buying and then collateral removal.

## Vulnerability Detail

Whenever Oracle reported rate is stale (`oracle.get(oracleData)` doesn't return an updated value), while market rate has dropped more than `FEE_PRECISION - collateralizationRate` (`FEE_PRECISION` scale), it is possible to atomically open borrow posiiton, buy collateral from the market and then remove extra collateral from the system with no prior positions and no investment.

The possibility of opening new positions, expecially leveraged ones, with a stale rate isn't required for BB or SGL core functionality, it constitutes a possible attack vector with very low business value of this possibility by itself.

## Impact

When a collateral can be bought from the market at a rate lower than Oracle reported stale rate by more than `FEE_PRECISION - collateralizationRate`, the difference between rate mismatch and this buffer can be extracted from the protocol by anyone with no prepositioning or investment needed.

The probability of such a drop combined with Oracle staleness can be estimated as low, but once this happens given the absense of barriers to entry the attack will be carried out with high probability. The impact itself if direct loss of protocol principal funds as bad debt will be atomically created this way, which has to be covered by other assets of the system thereafter.

Likelihood: Low + Impact: High = Severity: Medium.

## Code Snippet

`updateExchangeRate()` allows for stale rate within `rateValidDuration`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/Market.sol#L372-L385

```solidity
    function updateExchangeRate() public returns (bool updated, uint256 rate) {
        (updated, rate) = oracle.get(oracleData);

        if (updated) {
            require(rate != 0, "Market: invalid rate");
            exchangeRate = rate;
            rateTimestamp = block.timestamp;
            emit LogExchangeRate(rate);
        } else {
>>          require(rateTimestamp + rateValidDuration >= block.timestamp, "Market: rate too old");
            // Return the old rate if fetching wasn't successful & rate isn't too old
            rate = exchangeRate;
        }
    }
```

`solvent` check is used a the only control for a number of operations:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/Market.sol#L163-L170

```solidity
    modifier solvent(address from, bool liquidation) {
        updateExchangeRate();
        _accrue();

        _;

        require(_isSolvent(from, exchangeRate, liquidation), "Market: insolvent");
    }
```

Including atomic opening of the leveraged borrow position both in BB and SGL:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L53-L58

```solidity
    function buyCollateral(address from, uint256 borrowAmount, uint256 supplyAmount, bytes calldata data)
        external
        optionNotPaused(PauseType.LeverageBuy)
>>      solvent(from, false)
        notSelf(from)
        returns (uint256 amountOut)
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLeverage.sol#L47-L52

```solidity
    function buyCollateral(address from, uint256 borrowAmount, uint256 supplyAmount, bytes calldata data)
        external
        optionNotPaused(PauseType.LeverageBuy)
>>      solvent(from, false)
        notSelf(from)
        returns (uint256 amountOut)
```

## Tool used

Manual Review

## Recommendation

Consider introducing another level of control and allowing leverage operations and new positions opening only when the Oracle reported rate is current, e.g.:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/Market.sol#L372-L385

```diff
-   function updateExchangeRate() public returns (bool updated, uint256 rate) {
+   function updateExchangeRate(bool updateRequired) public returns (bool updated, uint256 rate) {
        (updated, rate) = oracle.get(oracleData);

        if (updated) {
            require(rate != 0, "Market: invalid rate");
            exchangeRate = rate;
            rateTimestamp = block.timestamp;
            emit LogExchangeRate(rate);
        } else {
-           require(rateTimestamp + rateValidDuration >= block.timestamp, "Market: rate too old");
+           require(!updateRequired && rateTimestamp + rateValidDuration >= block.timestamp, "Market: rate too old");
            // Return the old rate if fetching wasn't successful & rate isn't too old
            rate = exchangeRate;
        }
    }
```

`liquidation` flag can be dropped from `solvent` modifier since it is used only with `liquidation == false` (while `_isSolvent()` is being called directly on liquidations both in BB and SGL), and replaced with the `updateRequired` flag proposed, e.g.:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/Market.sol#L163-L170

```diff
-   modifier solvent(address from, bool liquidation) {
+   modifier solvent(address from, bool updateRequired) {
-       updateExchangeRate();
+       updateExchangeRate(updateRequired);
        _accrue();

        _;

-       require(_isSolvent(from, exchangeRate, liquidation), "Market: insolvent");
+       require(_isSolvent(from, exchangeRate, false), "Market: insolvent");
    }
```

All risk increase operations, i.e. leverage buying, borrow and collateral removal can utilize `solvent` modifiers with `updateRequired == true`, enforcing the Oracle reading to be current:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBLeverage.sol#L53-L58

```diff
    function buyCollateral(address from, uint256 borrowAmount, uint256 supplyAmount, bytes calldata data)
        external
        optionNotPaused(PauseType.LeverageBuy)
-       solvent(from, false)
+       solvent(from, true)
        notSelf(from)
        returns (uint256 amountOut)
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/bigBang/BBBorrow.sol#L37-L42

```diff
    function borrow(address from, address to, uint256 amount)
        external
        optionNotPaused(PauseType.Borrow)
        notSelf(to)
-       solvent(from, false)
+       solvent(from, true)
        returns (uint256 part, uint256 share)
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/origins/Origins.sol#L162-L165

```diff
    function removeCollateral(uint256 share)
        external
        optionNotPaused(PauseType.RemoveCollateral)
-       solvent(msg.sender, false)
+       solvent(msg.sender, true)
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/origins/Origins.sol#L175-L178

```diff
    function borrow(uint256 amount)
        external
        optionNotPaused(PauseType.Borrow)
-       solvent(msg.sender, false)
+       solvent(msg.sender, true)
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLBorrow.sol#L29-L34

```diff
    function borrow(address from, address to, uint256 amount)
        external
        optionNotPaused(PauseType.Borrow)
-       solvent(from, false)
+       solvent(from, true)
        notSelf(to)
        returns (uint256 part, uint256 share)
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLCollateral.sol#L48-L53

```diff
    function removeCollateral(address from, address to, uint256 share)
        external
        optionNotPaused(PauseType.RemoveCollateral)
-       solvent(from, false)
+       solvent(from, true)
        allowedBorrow(from, share)
        notSelf(to)
```

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/Tapioca-bar/contracts/markets/singularity/SGLLeverage.sol#L47-L52

```diff
    function buyCollateral(address from, uint256 borrowAmount, uint256 supplyAmount, bytes calldata data)
        external
        optionNotPaused(PauseType.LeverageBuy)
-       solvent(from, false)
+       solvent(from, true)
        notSelf(from)
        returns (uint256 amountOut)
```

All the other `solvent(from, false)` instances can stay intact.