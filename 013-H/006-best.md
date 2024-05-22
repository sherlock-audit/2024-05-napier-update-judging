Fierce Fern Python

high

# Users can deposit into tranche without any fee.

## Summary

Tranches receive some fees when users deposit into them. However, users can deposit into tranches without any fee by directly transferring underlying tokens into the `BaseLSTAdapter` right before deposit.

## Vulnerability Detail

Tranches receive some fees when users call to `Tranche.issue()`.  

https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/Tranche.sol#L163-L213

```solidity

    function issue(
        address to,
        uint256 underlyingAmount
    ) external nonReentrant whenNotPaused notExpired returns (uint256 issued) {
            [...]
@>      uint256 fee = underlyingAmount.mulDivUp(issuanceFeeBps, MAX_BPS);
            [...]
        _underlying.safeTransferFrom(msg.sender, feeRecipient, fee);
        _underlying.safeTransferFrom(msg.sender, address(adapter), underlyingAmount - fee);
@>      (, uint256 sharesMinted) = adapter.prefundedDeposit();
            [...]
    }

```
However, `sharesMinted` is calculated from the amount of WETH stored in the `BaseLSTAdapter`, not from `underlyingAmount - fee`. 

https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L80-L161

```solidity

    function prefundedDeposit() external nonReentrant onlyTranche returns (uint256, uint256) {
        uint256 bufferEthCache = bufferEth; // cache storage reads
        uint256 queueEthCache = totalQueueEth; // cache storage reads
@>      uint256 assets = IWETH9(WETH).balanceOf(address(this)) - bufferEthCache; // amount of WETH deposited at this time
        uint256 shares = previewDeposit(assets);
            [...]
        return (assets, shares);
    }

```
This makes it possible for users to avoid fee payment.

Consider the following scenario:
1. Alice transfers 10WETH to `BaseLSTAdapter`.
2. Alice calls `issue(to, 0)`.
Then the tranche cannot receive any fee from Alice.

## Impact

Users can deposit into tranche without any fee.

## Code Snippet

https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L80-L161

https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/Tranche.sol#L163-L213

## Tool used

Manual Review

## Recommendation

The mechanism for receiving fee should be improved.