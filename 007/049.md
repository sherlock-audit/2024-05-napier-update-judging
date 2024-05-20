Sweet Slate Cyborg

medium

# The scenario where the buffer in PufETHAdapter can be much larger than expected

## Summary
There is a scenario where the buffer value can exceed the `targetBufferEth` or even `pufETH` value in `ETH`, which would result in `ePufETH` becoming less valuable and earning fewer rewards.

## Vulnerability Detail
### PufETHAdapter
Let's consider the following scenario:
`Prerequisite`: `maxStakeLimit of PufETHAdapter` = 10000 ETH (value from tests folder), `max stETH stake limit` = 150000 ETH (from mainnet but can be changed)

1)User deposits 1 `WETH` and receives `Principal and Yield tokens`, and Tranche.sol `ePufETH` shares.
2)After some time, another user deposits 100 `WETH`, and the current stake limit of `stETH = 1 ETH`.
```solidity
uint256 stakeLimit = STETH.getCurrentStakeLimit();
        if (stakeAmount > stakeLimit) {
            // Cap stake amount
            stakeAmount = stakeLimit;
        }
```

This will result in only 1 WETH being deposited into the Puffer Vault, while the remaining WETH will increase the bufferEth value.

Variables after first depsot:
```solidity
buffer = 0.1 WETH
totalAssets = 1 WETH
```

Second deposit:
```solidity
// (1 WETH + 100 WETH) * 10% = 10.1 ETH
uint256 targetBufferEth = ((totalAssets() + assets) * targetBufferPercentage) / BUFFER_PERCENTAGE_PRECISION;

// 0.1 WETH + 100 WETH = 100.1 WETH
uint256 availableEth = bufferEthCache + assets;

// 100.1 WETH + 0 - 10.1 WETH = 90 WETH
stakeAmount = availableEth + queueEthCache - targetBufferEth;

// 100.1 WETH - 1 WETH = 99.1 WETH
bufferEth = (availableEth - stakeAmount).toUint128();
```

There would be 0.9 + 1 = 1.9 `WETH` in `pufETH`, and 99.1 `WETH` in the buffer.
In other words, 98% of all deposited `WETH` will be in the buffer, while only 2% will be earning rewards.

My point is that if the buffer is insufficient, then the buffer must increase. However, in this report, I highlighted an issue where depositors of `ePufETH` will start earning fewer rewards because significantly more WETH will be in the buffer than necessary.

Additionally, when `isStakingPaused is true`, all deposited WETH is stored in the buffer, and the ratio of buffer WETH to staked WETH will also change accordingly:
```solidity
if (targetBufferEth >= availableEth + queueEthCache || data.isStakingPaused()) {
            /// WRITE ///
            $.bufferEth = availableEth.toUint128();
            return (assets, shares);
        }
```

I understand that the next user who deposits `WETH` into the adapter will decrease the buffer variable, but we cannot say for certain when this will happen: whether it will be in 1 hour, 1 day, or 1 week. Therefore, in the mitigation steps, I recommended that the rebalancer should have the ability to stake excess buffer in Puffer.

## Impact
Users will earn rewards that are lower than expected.


## Code Snippet
[src/adapters/puffer/PufETHAdapter.sol#L69-L73](https://github.com/sherlock-audit/2024-05-napier-update/blob/main/napier-uups-adapters/src/adapters/puffer/PufETHAdapter.sol#L69-L73)

## Tool used

Manual Review

## Recommendation
There are two ways to avoid such a scenario:

1)You can implement `depositETH` from [PufferVaultV2](https://etherscan.io/token/0xD9A442856C234a39a81a089C06451EBAa4306a72), which would eliminate this situation since there would be no interaction with `stETH`, but directly with `ETH`.  (except for cases when staking was paused for some time)
2)Alternatively, implement a function that only the rebalancer can call to manage the excess `WETH` stored in the buffer variable by staking it in `Puffer`.
