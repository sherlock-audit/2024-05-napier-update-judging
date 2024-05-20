Dandy Sandstone Fish

medium

# Adapters revert when 0 shares are minted, making it impossible to deposit under certain conditions

## Summary

Users are unable to deposit into an Adapter in some situations due to the `_stake()` function reverting.

## Vulnerability Detail

The function `_stake()` in all of the in-scope adapters reverts if the amounts of minted shares of the targeted protocol is `0`.

Funds are deposited in an adapter via the [prefundedDeposit()](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L80) function, which internally calls `_stake()` by passing the amount to stake in the protocol, `stakeAmount`:

```solidity
    ...SNIP...
    uint256 stakeAmount;
    unchecked {
        stakeAmount = availableEth + queueEthCache - targetBufferEth;
    }

    if (stakeAmount > availableEth) {
@>      stakeAmount = availableEth;
    }

    ...SNIP...
@>   stakeAmount = _stake(stakeAmount); // stake amount can be 0
    ...SNIP...
```

The amount to stake in the protocol, `stakeAmount`, can be restricted to `availableEth`. If `availableEth`/`stakeAmount` is low enough (but not `0`) for the targeted protocol to mint `0` shares all of the adapters in-scope will revert by throwing an `InvariantViolation();` error:

- [UniEthAdapter](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/bedrock/UniETHAdapter.sol#L80)
- [EETHAdapter](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/etherfi/EETHAdapter.sol#L87)
- [RsETHAdapter](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L87)
- [PufETHAdapter](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/puffer/PufETHAdapter.sol#L84)
- [RenzoAdapter](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/renzo/RenzoAdapter.sol#L67)
- [RswETHAdapter](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/swell/RswETHAdapter.sol#L68)

## Impact

Users won't not be able to deposit funds if the `stakeAmount` is not enough to mint at least 1 share. The protocol genrally allows users to deposit both when `stakeAmount` is `0` and when the maximum deposit cap has been reached on the target protocol, which is incosistent with the behaviour outlined in this report.

A [similar finding](https://github.com/sherlock-audit/2024-01-napier-judging/issues/105) was disclosed in the previous Napier contest.

## Code Snippet

## Tool used

Manual Review

## Recommendation

The function `_stake()` in the adapters should ensure that the shares minted are at least `1` before actually depositing the funds. This might introduce a lot of overhead for the calculations, an alternative solution is to have the `_stake()` functions always return `0` if `stakeAmount` is lower than a certain (small) threshold:

```solidity
function _stake(uint256 stakeAmount) internal override returns (uint256) {
    if (stakeAmount < 1e6) return 0;
    ...SNIP...
}
```

If going for a different fix please note that the [EETHAdapter](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/etherfi/EETHAdapter.sol#L85) will actually revert on the internal call to `deposit()` if `0` shares are minted, instead of in the adapter.

