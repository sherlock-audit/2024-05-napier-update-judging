Stale Inky Reindeer

medium

# Loss of referral rewards like $SWELL and ezRENZO points

## Summary
Loss of referral rewards 
    1. $SWELL and 
    2. 10% of ezRENZO points

## Vulnerability Detail

https://docs.swellnetwork.io/swell/governance-and-tokenomics/swell-token

>Protocol growth incentivization through various liquidity mining, `referral` and airdrop schemes


https://app.renzoprotocol.com/referral
> You earn 10% of the points your friends make 

![image](https://github.com/sherlock-audit/2024-05-napier-update-ironsidesec/assets/162350329/f23a4979-0553-4422-80fc-c1c377b0ada9)

https://docs.renzoprotocol.com/docs/product/renzo-ezpoints
> Referral system: Receive extra ezPoints when you invite other users who deposit ETH.


1. on `RenzoAdapter._stake`  use `depositETH(_referralId)` instead of `depositETH(0)`

![image](https://github.com/sherlock-audit/2024-05-napier-update-ironsidesec/assets/162350329/d60b63d7-c994-4a75-8976-a2805e490609)

2. on Swell's `RswETHAdapter._stake`, use `depositWithReferral` instead of `deposit`

![image](https://github.com/sherlock-audit/2024-05-napier-update-ironsidesec/assets/162350329/aa3e4d3a-c4ef-402d-93e4-9a08d866a515)


## Impact
Loss of referral rewards like $SWELL and ezRENZO points

## Code Snippet
https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/swell/RswETHAdapter.sol#L66

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/renzo/RenzoAdapter.sol#L65
## Tool used

Manual Review

## Recommendation

Implement referral reward triggering implementations like 
1. on `RenzoAdapter._stake`  use `depositETH(_referralId)` instead of `depositETH(0)`

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/renzo/RenzoAdapter.sol#L65

```solidity
File: 2024-05-napier-update\napier-uups-adapters\src\adapters\renzo\RenzoAdapter.sol

61:     function _stake(uint256 stakeAmount) internal override returns (uint256) {
...
66:   >>>   RENZO_RESTAKE_MANAGER.depositETH{value: stakeAmount}(0); 
...
71:     }

```
2. on Swell's `RswETHAdapter._stake`, use `depositWithReferral` instead of `deposit`

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/swell/RswETHAdapter.sol#L66

```solidity
File: 2024-05-napier-update\napier-uups-adapters\src\adapters\swell\RswETHAdapter.sol

61:     function _stake(uint256 stakeAmount) internal override returns (uint256) {
62:         if (stakeAmount == 0) return 0;
...
66:   >>>   RSWETH.deposit{value: stakeAmount}(); 
...
71:     }

```
