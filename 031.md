Future Flint Leopard

medium

# The function `claimWithdrawal` in `EETHAdapter` is not restricted which may cause the eeETH scale to be decreased maliciously

## Summary

The function `claimWithdrawal` in `EETHAdapter` is not restricted which may cause the eeETH scale to be decreased maliciously.

## Vulnerability Detail

Per the contest page, the withdraw operations is restricted access if calling function cause a vault share price may change.

> For permissioned functions, please list all checks and requirements that will be made before calling the function.
withdraw
Restricted access if calling function cause a vault share price may change.


However, in EETHAdapter, the function claimWithdrawal is not  restricted which may cause the eeETH scale to be decreased maliciously.

https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/etherfi/EETHAdapter.sol#L56

```solidity
/// @dev note eeETH scale may be decreased as etherfi has the withdrawal fee.
function claimWithdrawal(uint256 _requestId) external override nonReentrant {
    // ...
    
}
```

## Impact

eeETH scale may be decreased maliciously.

## Code Snippet

https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/etherfi/EETHAdapter.sol#L56

## Tool used

Manual Review

## Recommendation

Add the onlyRebalancer modifier.