Suave Lime Otter

high

# `claimWithdrawal` deletes the entire `totalQueueETH`

## Summary
`claimWithdrawal` wrongfully deletes all of the accounted queued eth

## Vulnerability Detail
When withdrawing funds within the `uniETHAdapter`, first a withdrawal request is made to the `BEDROCK_STAKING` contract, and then after some time can be redeemed. As expected, multiple withdraw requests can be queued at the same time 
```solidity
    function requestWithdrawal(uint256 withdrawAmount, uint256 deadline) external nonReentrant onlyRebalancer {
        if (block.timestamp > deadline) revert TransactionTooOld();
        (uint256 queuedEth, uint256 _requestId) = _requestWithdrawal(withdrawAmount);
        if (queueWithdrawal[_requestId] != 0) revert WithdrawalPending();
        /// WRITE ///
        totalQueueEth += queuedEth.toUint128();
        queueWithdrawal[_requestId] = queuedEth;
    }
```

The problem is that when claiming a withdraw, all queued ETH is deleted. This is problematic, if not all requests are claimable, as `totalQueuedETH` will be reduced with more than what was received, leading to undervalued shares.

```solidity
    function claimWithdrawal(uint256 _requestId) external override nonReentrant onlyRebalancer {
        // Check whether the request is from this adapter.
        if (queueWithdrawal[_requestId] == 0) revert NoPendingWithdrawal(); // Request is not from this adapter, not exist or already claimed

        // Bedrock doesn't support claiming ETH per requestId.
        // Someone can donate ETH with `redeemContract:pay` and add claimable ETH.
        // So, here make sure that the debt has been deleted from the queue.
        (, uint256 debt) = BEDROCK_STAKING.checkDebt(_requestId);
        if (debt > 0) revert RequestNotFinalized(); // Not finalized yet

        /// INTERACT ///
        uint256 claimable = IRedeem(BEDROCK_STAKING.redeemContract()).balanceOf(address(this)); // donation + unstaked ETH
        uint256 balanceBefore = address(this).balance;
        IRedeem(BEDROCK_STAKING.redeemContract()).claim(claimable);
        uint256 claimed = address(this).balance - balanceBefore;

        /// WRITE ///
        delete queueWithdrawal[_requestId];
        delete totalQueueEth;
        bufferEth += claimed.toUint128();

        IWETH9(Constants.WETH).deposit{value: claimed}();
        emit ClaimWithdrawal(_requestId, claimed);
    }
```

## Impact
Wrong accounting, undervalued shares.

## Code Snippet
https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/bedrock/UniETHAdapter.sol#L109

## Tool used

Manual Review

## Recommendation
fix accounting 