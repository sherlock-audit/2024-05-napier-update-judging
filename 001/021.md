Stale Inky Reindeer

high

# Depositing `stETH` to puffer finance will revert due to wrong implementation of `PufETHAdapter._stake` call

## Summary
Reason: `PufETHAdapter._stake` will always revert due to wrong external call implementation.
Impact: Can't deposit to Puffer.
Likelihood: always.


## Vulnerability Detail

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/puffer/PufETHAdapter.sol#L82

```solidity
File: 2024-05-napier-update\napier-uups-adapters\src\adapters\puffer\PufETHAdapter.sol

66:     function _stake(uint256 stakeAmount) internal override returns (uint256) {
...
74: 
75:         IWETH9(Constants.WETH).withdraw(stakeAmount);
76:         uint256 _stETHAmt = STETH.balanceOf(address(this));
77:         STETH.submit{value: stakeAmount}(address(this));
78:         _stETHAmt = STETH.balanceOf(address(this)) - _stETHAmt;
79:         if (_stETHAmt == 0) revert InvariantViolation();
80: 
81:         // Stake stETH to PufferDepositor
82:    >>>  uint256 _pufETHAmt = PUFFER_DEPOSITOR.depositStETH(Permit(block.timestamp, _stETHAmt, 0, 0, 0));
84: 
...
88:     }

```

**Issue flow**:
1. When depositing by calling  `PUFFER_DEPOSITOR.depositStETH(Permit)`, `PufETHAdapter` passes only one parameter `Permit` look at line 82 above.
2. But the current `PUFFER_DEPOSITOR.depositStETH` has 2 parameters (Permit, address recipient). Check https://etherscan.io/address/0x4aA799C5dfc01ee7d790e3bf1a7C2257CE1DcefF#writeProxyContract#F1.
3. This is due to the proxy upgrade of `PUFFER_DEPOSITOR` from implementation v1 to implementation v2. 

To check upgares of `PUFFER_DEPOSITOR`, scroll on https://etherscan.io/address/0x4aA799C5dfc01ee7d790e3bf1a7C2257CE1DcefF#writeProxyContract 

![image](https://github.com/sherlock-audit/2024-05-napier-update-ironsidesec/assets/162350329/7473f701-d666-40d9-935e-bae897ddcc9e)

Previous implementation where it had only one param https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a#code#F1#L182

![image](https://github.com/sherlock-audit/2024-05-napier-update-ironsidesec/assets/162350329/3cf4acc6-417e-4ddc-9214-45afef53ac65)

Latest implementation has 2 params https://etherscan.io/address/0x8c9517a9e99c74cd072a118d3dc6b4f3217f8b9b#code#F1#L67

![image](https://github.com/sherlock-audit/2024-05-napier-update-ironsidesec/assets/162350329/0ff0e234-6832-4d96-af78-3b63a3520fdf)


## Impact
Depositing stETH to puffer finance is not possible with current  `PufETHAdapter`

## Code Snippet

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/puffer/PufETHAdapter.sol#L82

https://etherscan.io/address/0x8c9517a9e99c74cd072a118d3dc6b4f3217f8b9b#code#F1#L41

https://etherscan.io/address/0x4aA799C5dfc01ee7d790e3bf1a7C2257CE1DcefF#writeProxyContract

## Tool used

Manual Review

## Recommendation

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/puffer/PufETHAdapter.sol#L82

```diff
    function _stake(uint256 stakeAmount) internal override returns (uint256) {
...
        IWETH9(Constants.WETH).withdraw(stakeAmount);
        uint256 _stETHAmt = STETH.balanceOf(address(this));
        STETH.submit{value: stakeAmount}(address(this));
        _stETHAmt = STETH.balanceOf(address(this)) - _stETHAmt;
        if (_stETHAmt == 0) revert InvariantViolation();

        // Stake stETH to PufferDepositor
-       uint256 _pufETHAmt = PUFFER_DEPOSITOR.depositStETH(Permit(block.timestamp, _stETHAmt, 0, 0, 0));
+       uint256 _pufETHAmt = PUFFER_DEPOSITOR.depositStETH(Permit(block.timestamp, _stETHAmt, 0, 0, 0), address(this));

        if (_pufETHAmt == 0) revert InvariantViolation();

        return stakeAmount;
    }
```