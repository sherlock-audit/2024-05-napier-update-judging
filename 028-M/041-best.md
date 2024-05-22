Curly Infrared Peacock

high

# Requesting withdrawal makes it susceptible to attacks, resulting in burning more tokens and financial losses

## Summary

When redeeming from validators, the protocol passes the parameter maxToBurn with a value of `type(uint256).max`, effectively providing no slippage protection. This could result in users burning more xETH than necessary.

## Vulnerability Detail
In the `UniETHAdapter_requestWithdrawal()` function, the protocol calls `BEDROCK_STAKING.redeemFromValidators()` to redeem ETH. 
```solidity
  uint256 debtPrior = BEDROCK_STAKING.debtOf(address(this));
        BEDROCK_STAKING.redeemFromValidators({
            ethersToRedeem: withdrawAmount,
            maxToBurn: type(uint256).max,
            deadline: block.timestamp + 1
        });


```


The parameter `maxToBurn` is passed as `type(uint256).max`, which is used in `redeemFromValidators()` to validate `xETHToBurn < maxToBurn` to prevent burning excessive xETH for slippage protection purposes. 
https://etherscan.deth.net/address/0x4beFa2aA9c305238AA3E0b5D17eB20C045269E9d
```solidity
    function redeemFromValidators(uint256 ethersToRedeem, uint256 maxToBurn, uint256 deadline) external nonReentrant returns(uint256 burned) {
        _require(block.timestamp < deadline, "USR001");
        _require(ethersToRedeem % DEPOSIT_SIZE == 0, "USR005");
        _require(ethersToRedeem > 0, "USR005");

        uint256 totalXETH = IERC20(xETHAddress).totalSupply();
        uint256 xETHToBurn = totalXETH * ethersToRedeem / currentReserve();
        _require(xETHToBurn <= maxToBurn, "USR004");

        // NOTE: the following procdure must keep exchangeRatio invariant:
        // transfer xETH from sender & burn
        // uint256 ratio = _exchangeRatioInternal();           // RATIO GUARD BEGIN
        IMintableContract(xETHAddress).burnFrom(msg.sender, xETHToBurn);
        _enqueueDebt(msg.sender, ethersToRedeem);           // queue ether debts
        // assert(ratio == _exchangeRatioInternal());          // RATIO GUARD END

        // return burned 
        return xETHToBurn;
    }

```

However, passing `type(uint256).max` effectively does not provide slippage protection, potentially exposing users to sandwich attacks and burning excessive xETH.

Similarly, in the `UniETHAdapter.withdraw()` function, there should be a check for the minimum received ETH amount. This check is essential because there is no corresponding slippage protection in `BEDROCK_STAKING.instantSwap()`.
```solidity
    function withdraw(uint256 uniEthAmount) external nonReentrant onlyRebalancer {
        /// INTERACT ///
        uint256 balanceBefore = address(this).balance;
        BEDROCK_STAKING.instantSwap(uniEthAmount);
        uint256 received = address(this).balance - balanceBefore;

        IWETH9(Constants.WETH).deposit{value: received}();

        /// WRITE ///
        bufferEth += received.toUint128();
    }

```

```solidity
    function instantSwap(uint256 tokenAmount) external nonReentrant whenNotPaused {
        (uint256 maxEthersToSwap, uint256 maxTokensToBurn) = _instantSwapRate(tokenAmount);
        // _require(maxTokensToBurn > 0 && maxEthersToSwap > 0, "USR007");

        // uint256 ratio = _exchangeRatioInternal();               // RATIO GUARD BEGIN
        // transfer token from user and burn, substract ethers from pending ethers
        IMintableContract(xETHAddress).burnFrom(msg.sender, maxTokensToBurn);
        totalPending -= maxEthersToSwap;
        // assert(ratio == _exchangeRatioInternal());              // RATIO GUARD END

        // track balance change
        _balanceDecrease(maxEthersToSwap);

        // transfer ethers to users
        payable(msg.sender).sendValue(maxEthersToSwap);
    }


```

## Impact
Users may be susceptible to sandwich attacks, resulting in financial losses.



## Code Snippet
https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/bedrock/UniETHAdapter.sol#L155

## Tool used

Manual Review

## Recommendation
The recommended fix is to set an appropriate value.
