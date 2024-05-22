Sweet Slate Cyborg

medium

# DOS vulnerability in the _stake function in RsETHAdapter.sol

## Summary
There is a scenario where a user tries to deposit the minimum amount of ETH, which is accepted by `LRTDepositPool`, but the transaction simply fails.

## Vulnerability Detail
`LRTDepositPool` has a `minAmountToDeposit` = 0.0001 ETH. 
It's also important to pay attention to the check for `minAmountToDeposit` in the `LRTDepositPool` smart contract:
```solidity
function _beforeDeposit(
        address asset,
        uint256 depositAmount,
        uint256 minRSETHAmountExpected
    )
        private
        view
        returns (uint256 rsethAmountToMint)
    {
-->     if (depositAmount == 0 || depositAmount < minAmountToDeposit) {
            revert InvalidAmountToDeposit();
        }
        /// ...
}
```
So, if `depositAmount < minAmountToDeposit`, the transaction will fail, but in `RsETHAdapter`, there is an incorrect check that will not allow depositing the `minAmountToDeposit` value:
```solidity
if (stakeAmount <= RSETH_DEPOSIT_POOL.minAmountToDeposit()) revert MinAmountToDepositError();
```

I want to consider two cases:

1)If the user's `stakeAmount` is equal to `minAmountToDeposit`, the transaction will fail, although it should not. If the user specifies `stakeAmount > minAmountToDeposit`, the transaction will be successfully executed.
2)If `stakeAmount > stakeLimit and stakeLimit = minAmountToDeposit`, then `stakeAmount` will be set to `minAmountToDeposit`. This situation will prevent the user from making a deposit, and it can only be fixed by correcting the code:
```solidity
uint256 stakeLimit = RSETH_DEPOSIT_POOL.getAssetCurrentLimit(Constants.ETH); // 0.0001 ETH
        if (stakeAmount > stakeLimit) { // 1 ETH > 0.0001 ETH
            // Cap stake amount
            stakeAmount = stakeLimit;  // stakeAmount = 0.0001 ETH
        }
        // Check LRTDepositPool minAmountToDeposit
        if (stakeAmount <= RSETH_DEPOSIT_POOL.minAmountToDeposit()) revert MinAmountToDepositError(); // tx revert
```

## Impact
The user is unable to deposit the minimum amount of ETH that `LRTDepositPool` accepts.

## Code Snippet
[src/adapters/kelp/RsETHAdapter.sol#L77](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L77)

## Tool used

Manual Review

## Recommendation
Consider modifying the check for `minAmountToDeposit`:
```diff
- if (stakeAmount <= RSETH_DEPOSIT_POOL.minAmountToDeposit()) revert MinAmountToDepositError();
+ if (stakeAmount < RSETH_DEPOSIT_POOL.minAmountToDeposit()) revert MinAmountToDepositError();
```
