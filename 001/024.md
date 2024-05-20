Stale Inky Reindeer

medium

# Missing stake limit validation on `RenzoAdapter._stake`

## Summary

Every `_stake` function in adapter contracts like `RsETHAdapter`, `PufETHAdapter`, and `RenzoAdapter` has the below @dev comment to implement  the stake limit validation.

> /// @dev Need to check the current staking limit before staking to prevent DoS.

But only `RsETHAdapter`, `PufETHAdapter` validate the stake limits as shown below. But `RenzoAdapter` doesn't validate the stake limit and it reverts in an edge case.

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/puffer/PufETHAdapter.sol#L69-L73

```solidity
File: 2024-05-napier-update\napier-uups-adapters\src\adapters\puffer\PufETHAdapter.sol

65:  >>>   /// @dev Need to check the current staking limit before staking to prevent DoS.
66:     function _stake(uint256 stakeAmount) internal override returns (uint256) {
67:         if (stakeAmount == 0) return 0;
68: 
69:         uint256 stakeLimit = STETH.getCurrentStakeLimit();
70:   >>>   if (stakeAmount > stakeLimit) {
71:             // Cap stake amount
72:             stakeAmount = stakeLimit;
73:         }
```

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L71-L75

```solidity
File: 2024-05-napier-update\napier-uups-adapters\src\adapters\kelp\RsETHAdapter.sol

66:  >>>   /// @dev Need to check the current staking limit before staking to prevent DoS.
67:     function _stake(uint256 stakeAmount) internal override returns (uint256) {
68:         if (stakeAmount == 0) return 0;
69: 
70:         // Check LRTDepositPool stake limit
71:         uint256 stakeLimit = RSETH_DEPOSIT_POOL.getAssetCurrentLimit(Constants.ETH);
72:  >>>    if (stakeAmount > stakeLimit) {
73:             // Cap stake amount
74:             stakeAmount = stakeLimit;
75:         }

```

 
## Vulnerability Detail

1. `RenzoAdapter._stake` calls `depositETH` on `RENZO_RESTAKE_MANAGER`

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/renzo/RenzoAdapter.sol#L59

```solidity
File: 2024-05-napier-update\napier-uups-adapters\src\adapters\renzo\RenzoAdapter.sol

60:  >>>    /// @dev Need to check the current staking limit before staking to prevent DoS. 
61:     function _stake(uint256 stakeAmount) internal override returns (uint256) {
62:         if (stakeAmount == 0) return 0;
63:         if (RENZO_RESTAKE_MANAGER.paused()) revert ProtocolPaused();
64:         uint256 balanceBefore = EZETH.balanceOf(address(this));
65:         IWETH9(Constants.WETH).withdraw(stakeAmount);
66:         RENZO_RESTAKE_MANAGER.depositETH{value: stakeAmount}(0); // @audit-medium no referral id
67:         uint256 newBalance = EZETH.balanceOf(address(this));
68:         if (newBalance - balanceBefore == 0) revert InvariantViolation();
69: 
70:         return stakeAmount;

70:     }

```
2. And look at `depositETH` line highlighted with `>>>` below, it checks the `MaxTVLReached`, and it will revert if max TVL is reached. Maybe someone manipulated to cause DOS or unmanipulatedly hit the threshold triggering the revert. And the comment on `_stake` says to check the current limit to prevent DOS. But `RenzoAdapter._stake` is missing that.

https://etherscan.io/address/0xbaacd5f849024dcc80520baa952f11adfc59f9d0#code#F1#L558
Line 558 on https://etherscan.deth.net/address/0xbaacd5f849024dcc80520baa952f11adfc59f9d0

```solidity
    function depositETH(uint256 _referralId) public payable nonReentrant notPaused {
        // Get the total TVL
        (, , uint256 totalTVL) = calculateTVLs();

        // Enforce TVL limit if set
 >>>    if (maxDepositTVL != 0 && totalTVL + msg.value > maxDepositTVL) {
            revert MaxTVLReached();
        }

...
    }
```

## Impact

DOS or Missing validation the dev intended to make but didn't implement.

## Code Snippet

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/renzo/RenzoAdapter.sol#L59

https://etherscan.io/address/0xbaacd5f849024dcc80520baa952f11adfc59f9d0#code#F1#L558

Line 558 on https://etherscan.deth.net/address/0xbaacd5f849024dcc80520baa952f11adfc59f9d0

## Tool used

Manual Review

## Recommendation

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/renzo/RenzoAdapter.sol#L59

```diff
    function _stake(uint256 stakeAmount) internal override returns (uint256) {
        if (stakeAmount == 0) return 0;
        if (RENZO_RESTAKE_MANAGER.paused()) revert ProtocolPaused();

+       uint maxDepositTVL = RENZO_RESTAKE_MANAGER.maxDepositTVL();
+       uint totalTVL =  RENZO_RESTAKE_MANAGER.totalTVL();
+       if (maxDepositTVL != 0 && totalTVL + stakeAmount > maxDepositTVL) {
+           stakeAmount = maxDepositTVL - totalTVL;
+       }

        uint256 balanceBefore = EZETH.balanceOf(address(this));
        IWETH9(Constants.WETH).withdraw(stakeAmount);
        RENZO_RESTAKE_MANAGER.depositETH{value: stakeAmount}(0); // @audit-medium no referral id
        uint256 newBalance = EZETH.balanceOf(address(this));
        if (newBalance - balanceBefore == 0) revert InvariantViolation();

        return stakeAmount;
    }
```