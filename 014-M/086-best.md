Fast Vanilla Elephant

medium

# When there's a protocol-specific limit of the underlyingAmount that can be supplied through a single deposit, the remainder of the underlyingAmount that was charged from the user is not sent back to the user

## Summary
It's a common practice for a protocol to set a limit of amount that can be deposited once per a call.

Usually that is worth ~9000 WETH, but it depends on the protocol, and this is not a CONSTANT, i.e. it can be changed after the Napier adapter contracts are deployed and functioning.

## Vulnerability Detail
If the user decides to deposit a large amount AT ONCE, the real deposit that the Napier adapter will send to the protocol will be capped at the max limit that the protocol allows. This is usually fine, however there're no refunds that will send the user his funds back.

The protocol's limit is usually DYNAMIC, so under some conditions it can become a reaslistic and relatively small amount.

## Impact
User's funds will be stuck in the contract, and the real shares of a single 1 wei that the Adapter holds on the behalf of his users will be more expensive that in reality a 1 wei share is worth in the Kelp/PufETH pool.

## Code Snippet
Due to issues with running `forge test` in the `napier-uups-adapters` repo. (*No artifacts found for contract `Create2TrancheLib`...*), I'm not providing a PoC, unfortunately. Apologies for that.

But to give your an idea here I'm supplying the code snippets of where the problem comes from:

№1::

```solidity
// TrancheRouter.sol::issue
/// @notice deposit an `underlyingAmount` of underlying token into the yield source, receiving PT and YT.
    /// @dev Accept native ETH.
    /// @inheritdoc ITrancheRouter
    function issue(address adapter, uint256 maturity, uint256 underlyingAmount, address to)
        external
        payable
        nonReentrant
        returns (uint256)
    {
        ITranche tranche =
            TrancheAddress.computeAddress(adapter, maturity, TRANCHE_CREATION_HASH, address(trancheFactory));
        IERC20 underlying = IERC20(tranche.underlying());

        // Transfer underlying tokens to this contract
        // If this contract holds enough ETH, wrap it. Otherwise, transfer from the caller.
        if (address(underlying) == address(WETH9) && address(this).balance >= underlyingAmount) {
            WETH9.deposit{value: underlyingAmount}();
        } else {
            underlying.safeTransferFrom(msg.sender, address(this), underlyingAmount);
        }
        // Force approve
        underlying.forceApprove(address(tranche), underlyingAmount);

        return tranche.issue(to, underlyingAmount);
    }
```
↑ Here the funds are charged from the owner, *and notice that there're no prior checks whether this deposit will exceed the target protocol's deposit limit*.

The user may directly use the `Tranche` contract, but most users will use the router.

№2::

```solidity
// Tranche.sol::issue
function issue(
        address to,
        uint256 underlyingAmount
    ) external nonReentrant whenNotPaused notExpired returns (uint256 issued) {
        uint256 _lscale = lscales[to];
        uint256 _maxscale = gscales.maxscale;

        // NOTE: Updating mscale/maxscale in the cache before the issue to determine the accrued yield.
        uint256 cscale = adapter.scale();

        if (cscale > _maxscale) {
            // If the current scale is greater than the maxscale, update scales
            _maxscale = cscale;
            gscales.maxscale = cscale.toUint128();
        }

        // Deduct the issuance fee from the amount of underlying token deposited by the user
        // Fee should be rounded up towards the protocol (against the user) so that issued principal is rounded down
        // ```
        // fee = u * feeBps
        // shares = (u - fee) / s
        // ptIssued = shares * S
        // ```
        // where u = underlyingAmount, s = current scale and S = maxscale
        uint256 fee = underlyingAmount.mulDivUp(issuanceFeeBps, MAX_BPS);

        // Updating user's last scale to the latest maxscale
        lscales[to] = _maxscale;

        // If recipient has unclaimed interest, add it to the user's unclaimed yield.
        // Reminder: lscale is the last scale when the YT balance of the user was updated.
        if (_lscale != 0) {
            uint256 yBal = _yt.balanceOf(to);
            unclaimedYields[to] += _computeAccruedInterestInTarget(_maxscale, _lscale, yBal);
        }

        // Transfer fee to `feeRecipient` and the remaining underlying to the adapter
        _underlying.safeTransferFrom(msg.sender, feeRecipient, fee);
        _underlying.safeTransferFrom(msg.sender, address(adapter), underlyingAmount - fee); // the FULL amount is transferred, not accounting for any single-deposit limit caps
        (, uint256 sharesMinted) = adapter.prefundedDeposit();

        // Compute the amount of PT and YT to be minted
        // See above for the formula
        issued = sharesMinted.mulWadDown(_maxscale);

        // Mint PT and YT to user
        _mint(to, issued);
        _yt.mint(to, issued);

        emit Issue(msg.sender, to, issued, sharesMinted);
    }
```

↑ Here the funds are transferred from the router / the caller (if the user calls this contract directly), then the fee is decucted and the leftover is transferred to the target adapter.

Please notice that the full amount of the `underlyingToken` is transferred to the adapter, and no deposit limit caps are taken into account.

№3::

```solidity
// BaseLSTAdapterUpgradeable.sol

    /// @notice Handles prefunded deposits
    /// @return The amount of staked ETH
    /// @return The amount of shares minted
    function prefundedDeposit() external nonReentrant onlyTranche returns (uint256, uint256) {
        LSTAdapterStorage storage $ = _getStorage();

        uint256 bufferEthCache = $.bufferEth; // cache storage reads
        uint256 queueEthCache = $.totalQueueEth; // cache storage reads
        uint256 assets = IWETH9(WETH).balanceOf(address(this)) - bufferEthCache; // amount of WETH deposited at this time
        uint256 shares = previewDeposit(assets);

        if (assets == 0) return (0, 0);
        if (shares == 0) revert ZeroShares();

        // Calculate the target buffer amount considering the user's deposit.
        // bufferRatio is defined as the ratio of ETH balance to the total assets in the adapter in ETH.
        // Formula:
        // desiredBufferRatio = (totalQueueEth + bufferEth + assets - s) / (totalQueueEth + bufferEth + stakedEth + assets)
        // Where:
        // assets := Amount of ETH the user is depositing
        // s := Amount of ETH to stake at this time, s <= bufferEth + assets.
        //
        // Thus, the formula can be simplified to:
        // s = (totalQueueEth + bufferEth + assets) - (totalQueueEth + bufferEth + stakedEth + assets) * desiredBufferRatio
        //   = (totalQueueEth + bufferEth + assets) - targetBufferEth
        //
        // Flow:
        // If `s` <= 0, don't stake any ETH.
        // If `s` < bufferEth + assets, stake `s` amount of ETH.
        // If `s` >= bufferEth + assets, all available ETH can be staked in theory.
        // However, we cap the stake amount. This is to prevent the buffer from being completely drained.
        //
        // Let `a` be the available amount of ETH in the buffer after the deposit. `a` is calculated as:
        // a = (bufferEth + assets) - s
        uint256 targetBufferEth = ((totalAssets() + assets) * $.targetBufferPercentage) / BUFFER_PERCENTAGE_PRECISION;

        /// WRITE ///
        _mint(msg.sender, shares);

        uint256 availableEth = bufferEthCache + assets; // non-zero

        // If the buffer is insufficient or staking is paused, doesn't stake any of the deposit
        StakeLimitTypes.Data memory data = $.packedStakeLimitData.getStorageStakeLimitStruct();
        if (targetBufferEth >= availableEth + queueEthCache || data.isStakingPaused()) {
            /// WRITE ///
            $.bufferEth = availableEth.toUint128();
            return (assets, shares);
        }

        // Calculate the amount of ETH to stake
        uint256 stakeAmount; // can be 0
        unchecked {
            stakeAmount = availableEth + queueEthCache - targetBufferEth; // non-zero, no underflow
        }
        // If the calculated stake amount exceeds the available ETH, simply assign the available ETH to the stake amount.
        // Possible scenarios:
        // - Target buffer percentage was changed to a lower value and there is a large withdrawal request pending.
        // - There is a pending withdrawal request and the available ETH are not left in the buffer.
        // - There is no pending withdrawal request and the available ETH are not left in the buffer.
        if (stakeAmount > availableEth) {
            // Note: Admins should be aware of this situation and take action to refill the buffer.
            // - Pause staking to prevent further staking until the buffer is refilled
            // - Update stake limit to a lower value
            // - Increase the target buffer percentage
            stakeAmount = availableEth; // All available ETH
        }

        // If the amount of ETH to stake exceeds the current stake limit, cap the stake amount.
        // This is to prevent the buffer from being completely drained. This is not a complete solution.
        uint256 currentStakeLimit = StakeLimitUtils.calculateCurrentStakeLimit(data); // can be 0 if the stake limit is exhausted
        if (stakeAmount > currentStakeLimit) {
            stakeAmount = currentStakeLimit;
        }
        /// WRITE ///
        // Update the stake limit state in the storage
        $.packedStakeLimitData.setStorageStakeLimitStruct(data.updatePrevStakeLimit(currentStakeLimit - stakeAmount));

        /// INTERACT ///
        // Deposit into the yield source
        // Actual amount of ETH spent may be less than the requested amount.
        stakeAmount = _stake(stakeAmount); // stake amount can be 0

        /// WRITE ///
        $.bufferEth = (availableEth - stakeAmount).toUint128(); // no underflow theoretically

        return (assets, shares);
    }
```

↑ As you can see, the child (overriden) `_claim` function is called, but there's no prior accounting for a **SINGLE-DEPOSIT** depositing limit that the destination protocol might have.

And there're no refunds either. Though the user's minted shares correspond to the amount that was charged from the user in a proper manner, but the actual amount that will be deposited into the pool is lower than that, and the worth of a 1 wei share at the end will be less than the Adapter contract expects.

№4::

```solidity
// PufETHAdapter.sol
/// @notice Puffer allows stETH or wstETH via PufferDepositor.
    /// @dev Lido has a limit on the amount of ETH that can be staked.
    /// @dev Need to check the current staking limit before staking to prevent DoS.
    function _stake(uint256 stakeAmount) internal override returns (uint256) {
        if (stakeAmount == 0) return 0;

        uint256 stakeLimit = STETH.getCurrentStakeLimit();
        if (stakeAmount > stakeLimit) {
            // Cap stake amount
            stakeAmount = stakeLimit;
        }

        IWETH9(Constants.WETH).withdraw(stakeAmount);
        uint256 _stETHAmt = STETH.balanceOf(address(this));
        STETH.submit{value: stakeAmount}(address(this));
        _stETHAmt = STETH.balanceOf(address(this)) - _stETHAmt;
        if (_stETHAmt == 0) revert InvariantViolation();

        // Stake stETH to PufferDepositor
        uint256 _pufETHAmt = PUFFER_DEPOSITOR.depositStETH(Permit(block.timestamp, _stETHAmt, 0, 0, 0));

        if (_pufETHAmt == 0) revert InvariantViolation();

        return stakeAmount;
    }
```

↑

Here you can see the deposit amount is capped at the MAX amount that the `PUFFER_DEPOSITOR` allows per 1 single deposit.

Notice that this amount is NOT constant, and it can be changed by the admin or a permissioned role user of a protocol that the Adapter corresponds for.

![image](https://github.com/sherlock-audit/2024-05-napier-update-c-plus-plus-equals-c-plus-one/assets/105672704/94b864da-fbe9-40ac-a6d2-18c3fc7db660)


## Tool used

Manual Review

## Recommendation
Consider adding a mechanism of returning the user the remainder of a deposit if the **ACTUAL** deposited amount is **LESS** than the charged amount.

# References

https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/puffer/PufETHAdapter.sol#L63C5-L87C6

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-v1/src/Tranche.sol#L163C5-L213C6

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-v1/src/Tranche.sol#L202

https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/BaseLSTAdapterUpgradeable.sol#L158