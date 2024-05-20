Rhythmic Licorice Bear

high

# `MetapoolRouter#removeLiquidityOneETH()`'s attempt at removing liquidity could be completely bricked


## Summary

The [readMe](https://github.com/sherlock-audit/2024-05-napier-update/tree/c31af59c6399182fd04b40530d79d98632d2bfa7?tab=readme-ov-file#q-are-the-admins-of-the-protocols-your-contracts-integrate-with-if-any-trusted-or-restricted-if-these-integrations-are-trusted-should-auditors-also-assume-they-are-always-responsive-for-example-are-oracles-trusted-to-provide-non-stale-information-or-vrf-providers-to-respond-within-a-designated-timeframe) explicitly states that the Curve TriCrypto and TwoCrypto admins are **TRUSTED**, this means that it's expected and trusted of them to **immediately** call emergency exit on any pool Napier integrates with in the case where there is a black swan event pertaining that pool so as to ensure Napier's funds are safe... would be key to note that [Curve governance/admins have taken such actions multiple times](https://gov.curve.fi) in the [past](https://decrypt.co/56948/curve-finance-shuts-down-yv2-pool-after-finding-vulnerability) to ensure end users funds are safe.

Problem however is that Napier currently [hardcodes](https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/metapool-router/src/MetapoolRouter.sol#L423-L429) the method it uses in removing liquidity from the pool which works okay in normal instances, however this method wouldn't work if the trusted Curve admin takes the right/expected action to ensure Napier's funds are safe which would then cause the funds to be stuck.

## Vulnerability Detail

Take a look at https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/metapool-router/src/MetapoolRouter.sol#L401-L435

```solidity
    function removeLiquidityOneETH(
        address metapool,
        uint256 liquidity,
        uint256 minEthOut,
        address recipient,
        uint256 deadline
    ) external nonReentrant checkDeadline(deadline) checkMetapool(metapool) returns (uint256 ethOut) {
        // Steps:
        // If PT is matured, redemption of PT is allowed:
        // 1. Remove liquidity from the Curve metapool and withdraw one PT
        // 2. Redeem the PT for ETH

        // If PT is not matured: redemption of PT is not allowed yet:
        // 1. Remove liquidity from the Curve metapool and withdraw one base pool token
        // 2. Swap the received base pool token for ETH on the NapierPool

        ITranche pt = ITranche(Twocrypto(metapool).coins(PEGGED_PT_INDEX));

        SafeERC20.safeTransferFrom(Twocrypto(metapool), msg.sender, address(this), liquidity);

        if (block.timestamp >= pt.maturity()) {
            // If PT is matured, we can directly redeem the PT for ETH
|>             uint256 ptAmount = Twocrypto(metapool).remove_liquidity_one_coin(liquidity, PEGGED_PT_INDEX, 0);
            ethOut = pt.redeem({principalAmount: ptAmount, to: address(this), from: address(this)});
        } else {
            // Otherwise, redemption of PT is not allowed, so we need to swap the base pool token for ETH
|>            uint256 basePoolTokenAmount = Twocrypto(metapool).remove_liquidity_one_coin(liquidity, BASE_POOL_INDEX, 0);
            ethOut = triLSTPool.swapExactBaseLpTokenForUnderlying(basePoolTokenAmount, address(this));
        }

        if (minEthOut > ethOut) revert Errors.MetapoolRouterInsufficientETHOut();

        _unwrapWETH(recipient, ethOut);
    }

```

This is the only function that is used to remove liquidity from curve via the MetapoolRouter, and it is done in order to receive native ETH, evidently in both instances dependent on the maturity of the `PT` the `remove_liquidity_one_coin()` function is called to remove the liquidity. but this method only works when the pool has not been killed.

As explained under _summary_, when the trusted admin takes the action to ensure Napier's funds are safe, they would do this by killing the pool via the emergency action _`emergencyexit`_, which sets the ` self.is_killed` status of the pool to true and whenever` self.is_killed` in the curve pool contract becomes true all attempts to remove liquidity via `remove_liquidity_one_coin()` would always revert.

## Impact

Inability to remove liquidity.

## Code Snippet

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/metapool-router/src/MetapoolRouter.sol#L401-L435

## Tool used

Manual review

## Recommendation

When killed, it is only possible for existing LPs to remove liquidity via Curve's `remove_liquidity()` so consider integrating a functionality to query the pool's kill status and if it has indeed been killed then, reroute the `remove_liquidity_one_coin()` to instead query `remove_liquidity()`instead.
