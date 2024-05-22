Scruffy Iris Squirrel

medium

# Incorrect checking in `receiveFlashLoan` can cause `swapETHForYt` to fail unexpectedly.

## Summary
The following checking in `receiveFlashLoad` is incorrect, causing `swapETHForYt` to fail.
```solidity
File: metapool-router/src/MetapoolRouter.sol

333:        if (repayAmount > remaining) revert Errors.MetapoolRouterInsufficientETHRepay(); // Can't repay the flash loan
```

## Vulnerability Detail
In line 333, the check here is to ensure that the `ETH` sent by the user is enough to cover the `repayAmount`. Thus the check should revert if `repayAmount > TransientStorage.tloadU256(TSLOT_CB_DATA_VALUE)` , instead of `repayAmount > remaining`.

Let's assume that:
1. The user swaps `5 ETH` for some Yt by `swapETHForYt`. (i.e. `msg.value = 5 ETH`)
2. The estimated amount of WETH required to issue the PT and YT is `4 WETH` (i.e. `wethDeposit = 4 WETH`). 
3. The PT are finally swapped to `2 WETH` (i.e. `wethReceived = 2 WETH`). 
4. The `feeAmount` of flash loan is `0.1 WETH`.

Then:
1. `repayAmount = wethDeposit + feeAmounts[0] = 4 WETH + 0.1 WETH = 4.1 WETH`
2. `spent = repayAmount - wethReceived = 4.1 WETH - 2 WETH = 2.1 WETH`
3. `remaining = 5 WETH - spent = 5 WETH - 2.1 WETH = 2.9 WETH`

Line 333 in `receiveFlashLoan` reverts as `repayAmount > remaining`, and the `swapETHForYt` fails. But the user pays more than required to swap. The fail is not expected.
```solidity
File: metapool-router/src/MetapoolRouter.sol

323:        // Calculate the amount of ETH spent in the swap
324:        uint256 repayAmount = wethDeposit + feeAmounts[0];
325:        uint256 spent = repayAmount - wethReceived; // wethDeposit + feeAmounts[0] - wethReceived
326:
327:        // Revert if the ETH spent exceeds the specified maximum
328:        if (spent > TransientStorage.tloadU256(TSLOT_CB_DATA_MAX_ETH_SPENT)) {
329:            revert Errors.MetapoolRouterExceededLimitETHIn();
330:        }
331:
332:        uint256 remaining = TransientStorage.tloadU256(TSLOT_CB_DATA_VALUE) - spent;
333:@>      if (repayAmount > remaining) revert Errors.MetapoolRouterInsufficientETHRepay(); // Can't repay the flash loan
```
https://github.com/sherlock-audit/2024-05-napier-update/blob/main/metapool-router/src/MetapoolRouter.sol#L323-L333

## Impact
Incorrect checking can cause `swapETHForYt` to fail unexpectedly, breaking the core functionality of the contract.

## Code Snippet
https://github.com/sherlock-audit/2024-05-napier-update/blob/main/metapool-router/src/MetapoolRouter.sol#L323-L333

## Tool used

Manual Review

## Recommendation
```solidity
-        if (repayAmount > remaining) revert Errors.MetapoolRouterInsufficientETHRepay();
+        if (repayAmount > TransientStorage.tloadU256(TSLOT_CB_DATA_VALUE)) revert Errors.MetapoolRouterInsufficientETHRepay();
```
