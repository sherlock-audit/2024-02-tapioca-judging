Zealous Pineapple Duck

medium

# Balancer rebalance operation is permanently blocked whenever owner assigns `rebalancer` role to some other address

## Summary

Balancer's `rebalance()` controls access rights by requesting `msg.sender` to simultaneously be owner and `rebalancer`, which blocks it whenever this role is assigned to any other address besides owner's (that should be the case for production use).

## Vulnerability Detail

Balancer's core operation can be blocked due to structuring of the access control check, which requires `msg.sender` to have both roles instead of either one of them.

## Impact

Rebalancing, which is core functionality for mTOFT workflow, becomes inaccessible once owner transfers the `rebalancer` role elsewhere. To unblock the functionality the role has to be returned to the owner address and kept there, so rebalancing will have to be performed only directly from owner, which brings in operational risks as keeper operations will have to be run from owner account permanently, which can be compromised with higher probability this way.

Also, there is an impact of having `rebalancer` role set to a keeper bot and being unable to perform the rebalancing for a while until protocol will have role reassigned and the scripts run from owner account. This additional time needed can be crucial for user operations and in some situations lead to loss of funds.

Likelihood: Low + Impact: High = Severity: Medium.

## Code Snippet

Initially `owner` and `rebalancer` are set to the same address:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/Balancer.sol#L101-L110

```solidity
    constructor(address _routerETH, address _router, address _owner) {
        ...

        transferOwnership(_owner);
        rebalancer = _owner;
        emit RebalancerUpdated(address(0), _owner);
    }
```

Owner can then transfer `rebalancer` role to some other address, e.g. some keeper contract:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/Balancer.sol#L142-L149

```solidity
    /**
     * @notice set rebalancer role
     * @param _addr the new address
     */
    function setRebalancer(address _addr) external onlyOwner {
 >>     rebalancer = _addr;
        emit RebalancerUpdated(rebalancer, _addr);
    }
```

Once owner transfers `rebalancer` role to anyone else, it will be impossible to rebalance as it's always `(msg.sender != owner() || msg.sender != rebalancer) == true`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/Balancer.sol#L160-L176

```solidity
    /**
     * @notice performs a rebalance operation
     * @dev callable only by the owner
     * @param _srcOft the source TOFT address
     * @param _dstChainId the destination LayerZero id
     * @param _slippage the destination LayerZero id
     * @param _amount the rebalanced amount
     * @param _ercData custom send data
     */
    function rebalance(
        address payable _srcOft,
        uint16 _dstChainId,
        uint256 _slippage,
        uint256 _amount,
        bytes memory _ercData
    ) external payable onlyValidDestination(_srcOft, _dstChainId) onlyValidSlippage(_slippage) {
>>      if (msg.sender != owner() || msg.sender != rebalancer) revert NotAuthorized();
```

## Tool used

Manual Review

## Recommendation

Consider updating the access control to allow either owner or `rebalancer`, e.g.:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/Balancer.sol#L169-L176

```diff
    function rebalance(
        ...
    ) external payable onlyValidDestination(_srcOft, _dstChainId) onlyValidSlippage(_slippage) {
-       if (msg.sender != owner() || msg.sender != rebalancer) revert NotAuthorized();
+       if (msg.sender != owner() && msg.sender != rebalancer) revert NotAuthorized();
```
