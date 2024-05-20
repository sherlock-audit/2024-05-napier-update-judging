Virtual Brown Seal

medium

# Pssible DoS atack

## Summary

## Vulnerability Detail
The function [stake_](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/renzo/RenzoAdapter.sol#L60) doesn`t check the current stake limit
## Impact
It will lead to DoS atack
## Code Snippet
[RenzoAdapter.sol - (function _stake)](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/renzo/RenzoAdapter.sol#L60)
## Tool used

Manual Review

## Recommendation
Check the current  stake limit before staking like in [PufETHAdapter.sol - function _stake](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/puffer/PufETHAdapter.sol#L66)