Dandy Sandstone Fish

medium

# Kelp adapter won't allow users to deposit if `getAssetCurrentLimit` returns `0`

## Summary

Users will be unable to deposit into the Kelp adapter when 

## Vulnerability Detail

The function `_stake()` in the [RsETHAdapter::_stake()](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L67) checks the current limits on `ETH` deposits of the Kelp protocol before depositing:

```solidity
function _stake(uint256 stakeAmount) internal override returns (uint256) {
    if (stakeAmount == 0) return 0;

    uint256 stakeLimit = RSETH_DEPOSIT_POOL.getAssetCurrentLimit(Constants.ETH);
    if (stakeAmount > stakeLimit) {
        stakeAmount = stakeLimit;
    }
    ...SNIP..
}
```

The returned value of the call to `RSETH_DEPOSIT_POOL.getAssetCurrentLimit(Constants.ETH)` is assigned to the variable `stakeAmount`. This is the implementation of the called function, wich can be found [here](https://etherscan.io/address/0x036676389e48133B63a802f8635AD39E752D375D):

```solidity
function getAssetCurrentLimit(address asset) public view override returns (uint256) {
    uint256 totalAssetDeposits = getTotalAssetDeposits(asset);
    if (totalAssetDeposits > lrtConfig.depositLimitByAsset(asset)) {
        return 0;
    }

    return lrtConfig.depositLimitByAsset(asset) - totalAssetDeposits;
}
```

As it can be seen it's possible for the function to return `0`, which will make the subsequent calls in the `_stake()` function revert. This is inconsistent with the general behaviour of the software, which always returns `0` when `stakeAmount` is `0`.

## Impact

Users won't not be able to deposit funds in the `RsETHAdapter` because [RsETHAdapter::_stake()](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L67) will revert.

## Code Snippet

## Tool used

Manual Review

## Recommendation

In [RsETHAdapter::_stake()](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L68) move the `if (stakeAmount == 0) return 0;` line after the assets limits have been queried:

```solidity
function _stake(uint256 stakeAmount) internal override returns (uint256) {
    // Check LRTDepositPool stake limit
    uint256 stakeLimit = RSETH_DEPOSIT_POOL.getAssetCurrentLimit(Constants.ETH);
    if (stakeAmount > stakeLimit) {
        // Cap stake amount
        stakeAmount = stakeLimit;
    }

@>  if (stakeAmount == 0) return 0;

    // Check LRTDepositPool minAmountToDeposit
    if (stakeAmount <= RSETH_DEPOSIT_POOL.minAmountToDeposit()) return 0;
    // Check paused of LRTDepositPool
    if (RSETH_DEPOSIT_POOL.paused()) revert ProtocolPaused();

    // Interact
    IWETH9(Constants.WETH).withdraw(stakeAmount);
    uint256 _rsETHAmt = RSETH.balanceOf(address(this));
    RSETH_DEPOSIT_POOL.depositETH{value: stakeAmount}(0, REFERRAL_ID);
    _rsETHAmt = RSETH.balanceOf(address(this)) - _rsETHAmt;

    if (_rsETHAmt == 0) revert InvariantViolation();

    return stakeAmount;
}
```
[RsETHAdapter.sol#L77](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L68)
