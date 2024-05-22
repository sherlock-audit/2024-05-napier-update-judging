Sweet Slate Cyborg

medium

# PufETHAdapter does not handle the case when the stakeLimit of stETH is zero correctly

## Summary
`PufETHAdapter` does not handle the case when the `stakeLimit of stETH` is zero correctly.
```solidity
/// @dev Need to check the current staking limit before staking to prevent DoS.
    function _stake(uint256 stakeAmount) internal override returns (uint256) {
```

## Vulnerability Detail
If `STETH.getCurrentStakeLimit()` returns a zero value, then `stakeAmount` will also be 0, which will lead to a call to the `STETH.submit` function with zero `msg.value`, and the transaction will revert with `ZERO_DEPOSIT` reason.
```solidity
function _stake(uint256 stakeAmount) internal override returns (uint256) {
        if (stakeAmount == 0) return 0;

        uint256 stakeLimit = STETH.getCurrentStakeLimit();
        if (stakeAmount > stakeLimit) { // 1 WETH > 0
            // Cap stake amount
            stakeAmount = stakeLimit; // stakeAmount = 0
        }

        IWETH9(Constants.WETH).withdraw(stakeAmount);
        uint256 _stETHAmt = STETH.balanceOf(address(this));
        STETH.submit{value: stakeAmount}(address(this)); // tx revert
        
        ///code
}
```
The `submit` function of `stETH`:
```solidity
function _submit(address _referral) internal returns (uint256) {
        require(msg.value != 0, "ZERO_DEPOSIT");
```

## Impact
The transaction should not fail but instead increase the buffer on `availableEth`.

## Code Snippet
[src/adapters/puffer/PufETHAdapter.sol#L72](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/puffer/PufETHAdapter.sol#L72)

## Tool used

Manual Review

## Recommendation
Consider returning a zero value if `stakeLimit` equals 0:
```diff
uint256 stakeLimit = STETH.getCurrentStakeLimit();
        if (stakeAmount > stakeLimit) {
            // Cap stake amount
            stakeAmount = stakeLimit;
        }
+ if (stakeAmount == 0) return 0;        
```
