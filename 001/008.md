Abundant Daffodil Goldfish

medium

# `currentStakeLimit` depletes faster in some adapters, due to actual amount spent less than the input `stakeAmount`

## Summary
## Vulnerability Detail
In the `BaseLSTAdapterUpgradeable.prefundedDeposit()`, the stake amount is capped to the `currentStakeLimit`.This is to prevent the buffer from being completed drained. 

```solidity
      // Update the stake limit state in the storage
      $.packedStakeLimitData.setStorageStakeLimitStruct(data.updatePrevStakeLimit(currentStakeLimit - stakeAmount)); 
```

Before the staking occur, its checks whether the `stakeAmount` exceed current stakeLimit, if not modify the new stake limit to `currentStakeLimit - stakeAmount`. The issue is, the actual amount going to be spent could possibly lower than the `stakeAmount`

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/BaseLSTAdapterUpgradeable.sol#L157-L158

```solidity
        // Actual amount of ETH spent may be less than the requested amount.
        stakeAmount = _stake(stakeAmount); // stake amount can be 0
```

which means the stake limit that was updated previously does not account for the actual amount that we staked. I found one instance of adapters where this could possibly occur, 

**kelp/RsETHAdapter.sol**: Input `stakeAmount` modified to lower value if its [greater](https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L72-L75) than the stakeLimit of RsETHDeposit pool, 


## Impact
With every `prefundedDeposit` call where excess WETH is left to stake, the stake limit will deplete faster.
## Code Snippet
https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/BaseLSTAdapterUpgradeable.sol#L157
## Tool used
Manual Review
## Recommendation
The `_stake()` method do returns the actual spent amount, therefore I suggest to [update](https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/BaseLSTAdapterUpgradeable.sol#L153-L158) the staking limit after the staking has been done. 

```solidity
        /// INTERACT ///
        // Deposit into the yield source
        // Actual amount of ETH spent may be less than the requested amount.
        stakeAmount = _stake(stakeAmount); // stake amount can be 0

        /// WRITE ///
        // Update the stake limit state in the storage
        $.packedStakeLimitData.setStorageStakeLimitStruct(data.updatePrevStakeLimit(currentStakeLimit - stakeAmount));
```