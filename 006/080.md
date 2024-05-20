Rhythmic Shamrock Eel

medium

# `RsETHAdapter` uses an asset price that may not be up-to-date

## Summary

The `RsETHAdapter` contract calculates the total assets by summing the amount of ETH held and the value of the RsETH deposits using a price fetched from the KelpDAO oracle (`RSETH_ORACLE`). However, the price used is retrieved from a state variable in the oracle contract, which may not be up-to-date. This can lead to inaccurate asset valuation.

## Vulnerability Detail

The `RsETHAdapter` contract's `totalAssets()` function computes the total amount of ETH by combining the ETH held and the value of RsETH deposits. The RsETH value is derived by multiplying the RsETH balance with the price fetched from the oracle (`RSETH_ORACLE`). The issue is that the oracle's price is stored in a state variable, which requires an explicit update via `updateRSETHPrice()` to reflect the current value of the underlying assets and minted RsETH.

**Initialization of the Oracle in `RsETHAdapter`:**

```solidity
File: napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol
24:     /// @notice LRTOracle
25:     ILRTOracle constant RSETH_ORACLE = ILRTOracle(Constants.RSETH_ORACLE);
```

**`totalAssets` Function:**

```solidity
File: napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol
107:     function totalAssets() public view override returns (uint256) {
108:         LSTAdapterStorage storage $ = _getStorage();
109:         return $.totalQueueEth + $.bufferEth + (RSETH.balanceOf(address(this)) * RSETH_ORACLE.rsETHPrice()) / 1e18; //@audit price
110:     }
```

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L107

The `totalAssets()` function uses the `rsETHPrice()` function from the oracle, which returns the price stored in the contract's state. This price might be outdated if the `updateRSETHPrice()` function has not been called recently to refresh the price.

https://etherscan.io/address/0x8b9991f89fc31600dce064566cce28dc174fb8e4#code#F1#L66

## Impact

Using an outdated price can lead to inaccurate asset valuation, resulting in incorrect calculations for the total assets managed by the `RsETHAdapter`. This can affect the integrity of the protocol, potentially leading to incorrect user balances, mismanagement of funds, and loss of trust in the protocol.

## Code Snippet

Here is the problematic code in `RsETHAdapter`:

```solidity
File: napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol
107:     function totalAssets() public view override returns (uint256) {
108:         LSTAdapterStorage storage $ = _getStorage();
109:         return $.totalQueueEth + $.bufferEth + (RSETH.balanceOf(address(this)) * RSETH_ORACLE.rsETHPrice()) / 1e18; //@audit price
110:     }
```

## Tool used

Manual Review

## Recommendation

To ensure the price used for RsETH is up-to-date, call the `updateRSETHPrice()` function before retrieving the price. 
