Stale Inky Reindeer

high

# Less rsETH minted than intended in volatile conditions. due to zero slippage when staking ETH to mint rsETH

## Summary
This issue is not at all related to external admin, but the slippage parameter being zero is the issue. Read the issue flow after below images

## Vulnerability Detail
There is a slippage parameter called `minRSETHAmountExpected` in `RSETH_DEPOSIT_POOL.depositETH`.
And `RsETHAdapter._stake` is setting it to 0 when it calls `RSETH_DEPOSIT_POOL.depositETH`.

Its an issue, look at the sudden waves / bumps of rsETH within seconds, and LRT oracle admin will update the price according to the markets in dexes, or else  free arbitrage mints will further cause spiral moves.Or explore the ezETH depeg recently (https://x.com/SudipanSinha/status/1783473266515931284) and LRT x money market impact (https://x.com/SudipanSinha/status/1784107059744792715)


Click yellow button on https://www.dextools.io/app/en/ether/pair-explorer/0x059615ebf32c946aaab3d44491f78e4f8e97e1d3

![image](https://github.com/sherlock-audit/2024-05-napier-update-ironsidesec/assets/162350329/a42f8d41-7fa8-43d8-b2d1-3e12f876c4b0)

https://app.uniswap.org/explore/pools/ethereum/0x059615EBf32C946aaab3D44491f78e4F8e97e1D3

![image](https://github.com/sherlock-audit/2024-05-napier-update-ironsidesec/assets/162350329/721c378c-7727-43ad-953f-0ff56ba77119)

https://matcha.xyz/tokens/ethereum/0xa1290d69c65a6fe4df752f95823fae25cb99e5a7?buyChain=1&buyAddress=0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2

![image](https://github.com/sherlock-audit/2024-05-napier-update-ironsidesec/assets/162350329/b73d9bc5-d69e-4e2e-9823-21ddf294aa6a)


**Issue flow**: 
1. Rebalancer notices ETH to deposit in Kelp DAO's rsETH.
2. call flows to `RsETHAdapter._stake` --> `RSETH_DEPOSIT_POOL.depositETH` which will deposit eth to mint rsETH at current price.
3. Its a volatile time, and many transactions are pending, kelp lrt oracle is updated according to latest dex price which has pumped a lot.
4. Then the napier's `_stake`  transaction goes through, which will mint less rsETH because the price of rsETH went so high
5. but after few blocks, the price of rsETH again balanced and it dipped to normal, so lrt oracle is updated.

Now, it is a loss to Napier because we minted less tokens at peak price instead of normal current price and they are worth very low in terms of USD or WETH. If we used slippage, then it would have reverted in these cases.

Due to the below reason of price fluctuations, depegs, the slippage should not be set to 0.

And it is under the admin control. But during the times of rebalancing, if  depeg happens or in a very volatile sessions, the slippage is necessary. Because the admin might update the latest rsETH/WETH price and we might receive less rsETH as intended.

This can be resulted to MEV or sandwich attacks to 0 slippage. Although rebalancers use private mempool, the price of rsETH will be volatile in big steps wise jumps as shown in images below. And the latest Renzo  depeg of ezETH shows that slippage should be implemented, or else its a loss to Napier vaults.



https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L84

```solidity
File: 2024-05-napier-update\napier-uups-adapters\src\adapters\kelp\RsETHAdapter.sol

67:     function _stake(uint256 stakeAmount) internal override returns (uint256) {
...
81:         // Interact
82:         IWETH9(Constants.WETH).withdraw(stakeAmount);
83:         uint256 _rsETHAmt = RSETH.balanceOf(address(this));
84:   >>>   RSETH_DEPOSIT_POOL.depositETH{value: stakeAmount}(0, REFERRAL_ID); 
85:         _rsETHAmt = RSETH.balanceOf(address(this)) - _rsETHAmt;
86: 
...
90:     }

```

https://etherscan.io/address/0x13576cd2b61e601d3e98b5c06ef81896c9bbb369#code#F1#L206
Line 206 on https://etherscan.deth.net/address/0x13576cd2b61e601d3e98b5c06ef81896c9bbb369

```solidity
    function depositETH(
  >>>   uint256 minRSETHAmountExpected,
        string calldata referralId
    )
        external payable  whenNotPaused  nonReentrant
    {
        // checks
 >>>    uint256 rsethAmountToMint = _beforeDeposit(LRTConstants.ETH_TOKEN, msg.value, minRSETHAmountExpected);

        // interactions
        _mintRsETH(rsethAmountToMint);
        emit ETHDeposit(msg.sender, msg.value, rsethAmountToMint, referralId);
    }

    function _beforeDeposit(
        address asset,
        uint256 depositAmount,
 >>>    uint256 minRSETHAmountExpected
    )
        private view returns (uint256 rsethAmountToMint)
    {
... SKIP ...
        rsethAmountToMint = getRsETHAmountToMint(asset, depositAmount);

 >>>    if (rsethAmountToMint < minRSETHAmountExpected) {
            revert MinimumAmountToReceiveNotMet();
        }
    }
    
    function getRsETHAmountToMint(
        address asset,
        uint256 amount
    )
        public view  override   returns (uint256 rsethAmountToMint)
    {
        // setup oracle contract
        address lrtOracleAddress = lrtConfig.getContract(LRTConstants.LRT_ORACLE);
        ILRTOracle lrtOracle = ILRTOracle(lrtOracleAddress);

        // calculate rseth amount to mint based on asset amount and asset exchange rate
  >>>  rsethAmountToMint = (amount * lrtOracle.getAssetPrice(asset)) / lrtOracle.rsETHPrice();
    }
```

## Impact
Less rsETH was minted than intended in volatile conditions.

## Code Snippet

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L84

## Tool used

Manual Review

## Recommendation

https://github.com/sherlock-audit/2024-05-napier-update/blob/c31af59c6399182fd04b40530d79d98632d2bfa7/napier-uups-adapters/src/adapters/kelp/RsETHAdapter.sol#L84

```diff
-   function _stake(uint256 stakeAmount) internal override returns (uint256) {
+   function _stake(uint256 stakeAmount, uint256 minRSETHAmountExpected) internal override returns (uint256) {

... SKIP ...

        // Interact
        IWETH9(Constants.WETH).withdraw(stakeAmount);
        uint256 _rsETHAmt = RSETH.balanceOf(address(this));
-       RSETH_DEPOSIT_POOL.depositETH{value: stakeAmount}(0, REFERRAL_ID); 
+       RSETH_DEPOSIT_POOL.depositETH{value: stakeAmount}(minRSETHAmountExpected, REFERRAL_ID); 
        _rsETHAmt = RSETH.balanceOf(address(this)) - _rsETHAmt;

        if (_rsETHAmt == 0) revert InvariantViolation();

        return stakeAmount;
    }
```
