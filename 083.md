Great Seafoam Reindeer

medium

# No slippage check in `swapETHForYt` function  can lead to slippage losses during swap

## Summary
No slippage check in `swapETHForYt` function  can lead to slippage losses during swap
## Vulnerability Detail
```javascript
function swapETHForYt(
        address metapool,
@>        uint256 ytAmount,
@>        uint256 maxEthSpent,
        address recipient,
        uint256 deadline,
        ApproxParams calldata approx
    ) external payable nonReentrant checkDeadline(deadline) checkMetapool(metapool) 
```
The ytAmount is the amount of YT tokens that user wants to receive. But the acture amount of YT tokens that user will receive is pyIssued.
```javascript
function receiveFlashLoan(
        IERC20[] calldata, /* tokens */
        uint256[] calldata amounts,
        uint256[] calldata feeAmounts,
        bytes calldata /* userData */
    ) external {
        // CHECK
        // Note: Call only through `swapETHForYt` && from the Vault should be allowed.
        // This ensures that the function call is invoked by `swapETHForYt` entry point.
        // Checking `msg.sender == address(vault)` may not be sufficient as the call may be initiated by other contracts and pass arbitrary data.
        assembly {
            let ctx := tload(TSLOT_0)
            tstore(TSLOT_0, 0) // Delete the authorization (flag=address(0))
            if iszero(eq(ctx, address())) {
                mstore(0x00, 0x5c501941) // `MetapoolRouterUnauthorized()`
                revert(0x1c, 0x04)
            }
        }

        address pt = TransientStorage.tloadAddress(TSLOT_CB_DATA_PT);
        address metapool = TransientStorage.tloadAddress(TSLOT_CB_DATA_METAPOOL);

        // Issue PT tokens using the WETH
        if (_isApproved(address(WETH9), pt) == 0) {
            _setApproval(address(WETH9), pt);
            WETH9.approve(pt, type(uint256).max);
        }
        uint256 wethDeposit = amounts[0];
        uint256 pyIssued = ITranche(pt).issue(address(this), wethDeposit);

        // Swap the PT for the base pool token on the Curve metapool
        ITranche(pt).transfer(metapool, pyIssued);
        uint256 basePoolTokenOut =
@>            Twocrypto(metapool).exchange_received(PEGGED_PT_INDEX, BASE_POOL_INDEX, pyIssued, 0, address(this));

        // Swap the received base pool token for ETH on the NapierPool
@>        uint256 wethReceived = triLSTPool.swapExactBaseLpTokenForUnderlying(basePoolTokenOut, address(this));

        // Unreasonable situation: Received more WETH than sold
        if (wethReceived > wethDeposit) revert Errors.MetapoolRouterNonSituationSwapETHForYt();

        // Calculate the amount of ETH spent in the swap
        uint256 repayAmount = wethDeposit + feeAmounts[0];
        uint256 spent = repayAmount - wethReceived; // wethDeposit + feeAmounts[0] - wethReceived

        // Revert if the ETH spent exceeds the specified maximum
        if (spent > TransientStorage.tloadU256(TSLOT_CB_DATA_MAX_ETH_SPENT)) {
            revert Errors.MetapoolRouterExceededLimitETHIn();
        }

        uint256 remaining = TransientStorage.tloadU256(TSLOT_CB_DATA_VALUE) - spent;
        if (repayAmount > remaining) revert Errors.MetapoolRouterInsufficientETHRepay(); // Can't repay the flash loan

        // Temporarily store a return value of `swapETHForYt` function across the call context
        assembly {
            tstore(TSLOT_1, spent)
        }

        // Transfer the YT tokens to the recipient
@>        IERC20(ITranche(pt).yieldToken()).transfer(TransientStorage.tloadAddress(TSLOT_CB_DATA_RECEIPIENT), pyIssued);

        // Repay the flash loan
        WETH9.transfer(msg.sender, repayAmount);

        // Unwrap and send the remaining WETH back to the sender
        _unwrapWETH(TransientStorage.tloadAddress(TSLOT_CB_DATA_SENDER), remaining);
    }
```
Throughout the entire swap process, involving token swaps across multiple pools, the slippage protection parameter was not used. The maxEthSpent only guarantees the maximum amount of ETH spent, but it does not guarantee the minimum amount of YT tokens received.

## Impact
 Leading to slippage losses during swap
## Code Snippet
https://github.com/sherlock-audit/2024-05-napier-update/blob/main/metapool-router/src/MetapoolRouter.sol#L212C1-L266C6
https://github.com/sherlock-audit/2024-05-napier-update/blob/main/metapool-router/src/MetapoolRouter.sol#L282C5-L348C6
## Tool used

Manual Review

## Recommendation
Adding  the ytMinimum to ensure that the minimum amount of YT tokens must be received
