Curly Infrared Peacock

medium

# Exchanging uniETH for WETH in the Uniswap V3 pool incurs significant slippage

## Summary
The protocol exchanges uniETH for WETH in the Uniswap V3 uniETH/ETH pool, which has significantly less liquidity compared to the Curve uniETH/ETH pool. As a result, there is a higher slippage when exchanging in this pool.

## Vulnerability Detail

The purpose of the `UniETHAdapter.swapUniETHForETH()` function is to swap uniETH for ETH.

```solidity
    function swapUniETHForETH(
        uint256 amount,
        uint256 deadline,
        uint256 minEthOut,
        bytes calldata data
    ) external nonReentrant onlyRebalancer {
        if (amount >= 32 ether) revert SwapAmountTooLarge();

        /// INTERACT ///
        uint256 balanceBefore = IWETH9(Constants.WETH).balanceOf(address(this));
        SafeERC20.forceApprove(IERC20(Constants.UNIETH), address(swapper), amount + 1); // avoild storage value goes to 0
        swapper.swap(amount, deadline, minEthOut, data);
        uint256 received = IWETH9(Constants.WETH).balanceOf(address(this)) - balanceBefore;

        /// WRITE ///
        bufferEth += SafeCast.toUint128(received);
    }

```


 In this function, the protocol calls `swapper.swap()` to exchange uniETH for ETH, which occurs on the Uniswap V3 pool.
```solidity
    function swap(uint256 amountIn, uint256 deadline, uint256 minEthOut, bytes calldata data) external {
        SafeERC20.safeTransferFrom(IERC20(Constants.UNIETH), msg.sender, address(this), amountIn);
        // Uniswap V3 uniETH/ETH 0.05 % fee tier pool: https://app.uniswap.org/explore/tokens/ethereum/0xf1376bcef0f78459c0ed0ba5ddce976f1ddf51f4
        router.exactInputSingle(
            ISwapRouter.ExactInputSingleParams({
                tokenIn: Constants.UNIETH,
                tokenOut: Constants.WETH,
                fee: 500, // 0.05%
                recipient: msg.sender,
                deadline: deadline,
                amountIn: amountIn,
                amountOutMinimum: minEthOut,
                sqrtPriceLimitX96: abi.decode(data, (uint160))
            })
        );
    }

```


We can observe that the TVL for the Uniswap V3 uniETH/ETH pool is 364,000, and the daily trading volume is 28,000. 

https://app.uniswap.org/explore/tokens/ethereum/0xf1376bcef0f78459c0ed0ba5ddce976f1ddf51f4

On the other hand, the TVL for the Curve uniETH/ETH pool is 4,294,597.08, with a daily trading volume of 204,431.

https://curve.fi/#/ethereum/pools/factory-stable-ng-71/deposit

The liquidity in the Curve pool is better compared to the Uniswap V3 pool, resulting in smaller slippage. Therefore, exchanging on Uniswap V3 may incur higher slippage compared to the Curve pool.

## Impact
Exchanging uniETH for WETH in the Uniswap V3 uniETH/ETH pool incurs significant slippage. 

## Code Snippet
https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/bedrock/UniETHSwapper.sol#L27-L42
## Tool used

Manual Review

## Recommendation
The recommended fix is to use the Curve uniETH/ETH pool instead.
