Suave Lime Otter

high

# `uniETH` will get stuck within `uniETHSwapper` if swap crosses set `sqrtPriceLimitX96`

## Summary
Funds will get stuck within the `uniETHSwapper` contract. 

## Vulnerability Detail
When swapping `uniETH` for eth, the `rebalancer` inputs a `data` in which the `sqrtPriceLimitX96` is encoded.
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
                sqrtPriceLimitX96: abi.decode(data, (uint160))  // @audit - here
            })
        );
    }
```

The problem is that if the swap crosses that price, it stops. If 1 ETH is to be swapped, and only half of it can be swapped within the price limit, the router will only take that 0.5 eth and the rest will remain within the `uniETHSwapper` contract stuck 

Code snippet from the univ3 pool:
```solidity
        // continue swapping as long as we haven't used the entire input/output and haven't reached the price limit
        while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != sqrtPriceLimitX96) {
```

## Impact
Lost/ Stuck funds

## Code Snippet
https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-v1/src/adapters/bedrock/UniETHSwapper.sol#L39
https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L640
## Tool used

Manual Review

## Recommendation
refund any excess uniETH upon swap
