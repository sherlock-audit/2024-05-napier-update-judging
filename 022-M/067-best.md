Sweet Slate Cyborg

medium

# Issue with staking being paused and LRTDepositPool limit in RsETHAdapter.sol

## Summary
`RsETHAdapter` does not stake any of the deposit but increases the buffer variable when `isStakingPaused is true`. This can lead to a situation where the `WETH` tokens in the buffer exceed the current staking limit in `LRTDepositPool`, resulting in `eRsETH` becoming less valuable and earning fewer rewards.

## Vulnerability Detail
Currently, the maximum amount of ETH that all users can deposit in `Kelp DAO` is 100,000 ETH.
Let's consider a scenario:
Assume that the current staking limit of `LRTDepositPool` is 1 ETH.

1)The owner calls `pauseStaking` in `RsETHAdapter.sol`.
2)When users stake `WETH` while staking is paused, the bufferEth value increases. For example, during the period when staking was paused, the `bufferEth` increased by 10 ETH.
```solidity
if (targetBufferEth >= availableEth + queueEthCache || data.isStakingPaused()) {
            /// WRITE ///
            $.bufferEth = availableEth.toUint128();
            return (assets, shares);
        }
```
3)From this, it can be understood that a significant portion of WETH cannot be deposited in `LRTDepositPool`, which will result in all depositors starting to earn fewer rewards.

## Impact
Users will earn rewards that are lower than expected.

## Code Snippet
[src/adapters/BaseLSTAdapterUpgradeable.sol#L121-L125](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/BaseLSTAdapterUpgradeable.sol#L121-L125)

## Tool used

Manual Review

## Recommendation

It's difficult to say how to best fix this, as this limit would need to be checked in the `prefundedDeposit` function. However, since the contract is paused, `RSETH_DEPOSIT_POOL limit` can change at any time which would not solve the issue.
