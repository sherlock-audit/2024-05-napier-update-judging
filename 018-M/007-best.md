Fierce Fern Python

medium

# There is no mechanism for distributing some additional rewards including eigenlayer restaked points.

## Summary

eETH/weETH holders earns eigenlayer restaked points, as ether.fi will be restaking on EigenLayer. However, there is no mechanism for handling these additional rewards.

## Vulnerability Detail

EigenLayer Restaked Points offer additional benefits for those who contribute to the protocol’s security through restaking. These points can be seen as a measure of your contribution to EigenLayer’s restaking ecosystem, translating into further rewards and incentives.

Ether.fi says, "eETH/weETH holders will earn EigenLayer points in addition to ether.fi Loyalty Points, as ether.fi will be restaking on EigenLayer. The protocol will be remitting 100% of these points received to eETH holders from day 1, and displaying stakers allocation of the EigenLayer points in their dashboard."

Bedrock says, "Restaked points are a measure of your contribution to the shared security of the EigenLayer ecosystem. They represent staking participation equal to the time-integrated amount staked.
For example, a user who stakes 1 ETH for 1 hour accrues 1 restaked point.
Bedrock distributes 100% of the restaking points earned by its users' deposits"

Also, some LST protocols have their own points system.

In this protocal, the eETH/weETH holder is the `EEtherAdapter` and the UniETH holder is the `UniETHAdapter`.

However, there is no mechanism for distributing these points in the `LSTAdapters` such as `EEtherAdapter` and `UniETHAdapter`.

https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/etherfi/EETHAdapter.sol#L26-L137
```solidity
contract EEtherAdapter is BaseLSTAdapter, IERC721Receiver {
[...]
}
```

## Impact

Some additional rewards including eigenlayer restaked points cannot be taken.

## Code Snippet

https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/etherfi/EETHAdapter.sol#L26-L137

## Tool used

Manual Review

## Recommendation

A mechanism for taking additional rewards should be built in `LSTAdapters`.