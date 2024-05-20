Sticky Rouge Narwhal

medium

# MetapoolRouter.swapETHForPt() reverts due to a typo in a function parameter

## Summary
There is a typo in a function call in `MetapoolRouter.swapETHForPt()`. As a result, `MetapoolRouter.swapETHForPt()` reverts and doesn't work.

## Vulnerability Detail
`MetapoolRouter.swapETHForPt()` swaps WETH for the base pool token of NapierPool.

```solidity
MetapoolRouter.sol

143:         ethSpent = triLSTPool.swapUnderlyingForExactBaseLpToken({baseLpOut: basePoolTokenAmount, recipient: metapool});

```

`NapierPool.swapUnderlyingForExactBaseLpToken()` accepts `baseLptOut` parameter instead of `baseLpOut`.
```solidity
NapierPool.sol
459:     function swapUnderlyingForExactBaseLpToken(uint256 baseLptOut, address recipient)

```

As a result, `MetapoolRouter.swapETHForPt()` will revert and it doesn't work at all.


## Impact
`MetapoolRouter.swapETHForPt()` doesn't work.

## Tool used

Manual Review

## Code Snippet
https://github.com/sherlock-audit/2024-05-napier-update/tree/main/metapool-router/src/MetapoolRouter.sol#L143

https://github.com/sherlock-audit/2024-05-napier-update/blob/main/metapool-router/lib/v1-pool/src/NapierPool.sol#L459

## Recommendation
Use the correct parameter `baseLptOut` instead of `baseLpOut`.