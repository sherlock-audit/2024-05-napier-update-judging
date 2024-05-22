Curly Infrared Peacock

medium

# The redeem contract have Critical Risk

## Summary
The redeem contract has been reported by a large number of users for phishing and misleading users. Integrating with it may pose security risks or compliance issues, ultimately resulting in user loss.


## Vulnerability Detail
In the `UniETHAdapter.claimWithdrawal()` function, the protocol calls `Redeem(BEDROCK_STAKING.redeemContract()).claim()` to claim all claimable ETH. 

```solidity
  /// INTERACT ///
        uint256 claimable = IRedeem(BEDROCK_STAKING.redeemContract()).balanceOf(address(this)); // donation + unstaked ETH
        uint256 balanceBefore = address(this).balance;
        IRedeem(BEDROCK_STAKING.redeemContract()).claim(claimable);
        uint256 claimed = address(this).balance - balanceBefore;

```


According to the `Constants` contract, we found that the address for BEDROCK_STAKING is 0x4beFa2aA9c305238AA3E0b5D17eB20C045269E9d. 
```solidity
// @notice BedRock Staking address on mainnet
address constant BEDROCK_STAKING = 0x4beFa2aA9c305238AA3E0b5D17eB20C045269E9d;

```

From this, we determined that the `redeemContract` address is 0x98169228cB99Ed26c1043eD8Ca53A5Cb371D3B8D. 
https://etherscan.io/address/0x4beFa2aA9c305238AA3E0b5D17eB20C045269E9d#readProxyContract#F36

This address is a proxy contract with an implementation contract at 0x9ca778C263CfAe78a5F7d10F12E6BE25cB3C5f8C. 
https://etherscan.io/address/0x98169228cB99Ed26c1043eD8Ca53A5Cb371D3B8D#readProxyContract

However, this address has been reported by many users for phishing and is tagged with "Critical Risk." Therefore, this address poses a security risk or compliance issues, and integrating with it could lead to user loss due to its sensitivity.
`
 This token is reported to have been used for misleading people into believing it was sent from well-known addresses and may be spam or phishing. Please treat it with caution.
`
https://etherscan.io/address/0x9ca778c263cfae78a5f7d10f12e6be25cb3c5f8c#code
![image](https://github.com/sherlock-audit/2024-05-napier-update-sleepriverfish/assets/6247627/d9963098-443b-4b42-a948-594281fe2f30)


## Impact
It could lead to user loss due to its sensitivity.

## Code Snippet
https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/bedrock/UniETHAdapter.sol#L102

## Tool used

Manual Review

## Recommendation
It is recommended to exercise caution when integrating with this contract.