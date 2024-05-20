Dandy Sandstone Fish

medium

# Users can frontrun LSTs/LRTs tokens prices decrease in order to avoid losses

## Summary

Users can redeem their `PT`/`YT` tokens before a price decrease of a supported LST/LRT token in order to avoid losses.

## Vulnerability Detail

Napier allows users to redeem their `PT`/`YT` tokens for `ETH` via [BaseLSTAdapter::prefundedRedeem()](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L168) instantly if the amount to be withdrawn is lower or equal than the available `ETH` buffer. The in-scope adapters that allow this are:

- [EETHAdapter](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/etherfi/EETHAdapter.sol)
- [UniEthAdapter](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/bedrock/UniETHAdapter.sol)

A Napier user that staked in one of these adapters can:

1. Monitor the mempool and the beacon chain to know in advance if either the `eETH` or `uniETH` tokens will lose value.
2. Frontrun the value loss by redeeming their `PT` and `YT`tokens via [Tranche::redeemWithYT()](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/Tranche.sol#L231), which will call [BaseLSTAdapter::prefundedRedeem()](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L168), in exchange for `ETH`.

Because the value drop is still not reflected in the Napier protocol the staker will be able to withdraw his funds without being affected by the losses.

In the case of `eETH`, a rebase token, an attacker can know if a balance drop will happen by monitoring the mempool for calls to `rebase()` in the EtherFi [LiquidityPool](https://etherscan.io/address/0x308861A430be4cce5502d0A12724771Fc6DaF216#writeProxyContract) contract.

In the case of `uniEth` an attacker can know if the token will lose value by monitoring the protocol validators for penalties and slashing events. Bedrock (`uniEth`) is built on top of Eigenlayer, which can be notified of balance drops due to penalties or slashings via two permissionless functions: [EigenPod::verifyBalanceUpdates()](https://github.com/Layr-Labs/eigenlayer-contracts/blob/v0.2.1-goerli-m2/src/contracts/pods/EigenPod.sol#L185) and [EigenPod::verifyAndProcessWithdrawals()](https://github.com/Layr-Labs/eigenlayer-contracts/blob/v0.2.1-goerli-m2/src/contracts/pods/EigenPod.sol#L232). This allows an attacker to perform the following series of calls atomically to avoid losses:

1. Monitor the Bedrock validators on the beacon chain for penalties and slashings.
2. Call [Tranche::redeemWithYT()](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/Tranche.sol#L231) to redeem `PT`/`YT` in exchange of `ETH`.
3. Call [EigenPod::verifyBalanceUpdates()](https://github.com/Layr-Labs/eigenlayer-contracts/blob/v0.2.1-goerli-m2/src/contracts/pods/EigenPod.sol#L185)/[EigenPod::verifyAndProcessWithdrawals()](https://github.com/Layr-Labs/eigenlayer-contracts/blob/v0.2.1-goerli-m2/src/contracts/pods/EigenPod.sol#L232) to notify Eigenlayer of the balance drop.
4. The value of `uniETH` will instantly drop.
5. Deposit the previously withdrawn `ETH` for more `YT`/`PT` tokens than the initial amount.

Another instance that instantly lowers the value held by the `UniEthAdapter` adapter is the call to [UniETHAdapter::swapUniETHForETH()](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/bedrock/UniETHAdapter.sol#L185) because a `0.05%` fee is paid to UniswapV3, this can also be front run by stakers to avoid bearing the losses of the fee.

## Impact

Stakers can avoid losses, which implies honest stakers will lose more than they should.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Introduce a withdraw queue, this will prevent this kind of frontrunning attacks.

