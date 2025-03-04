# CRITICAL

# HIGH

# MEDIUM

## No checks if liquidity to mint is zero

### Description

1. In the function [`ArrakisV2.mint()`](https://github.com/ArrakisFinance/v2-core/blob/a9759d1a45bc3a9dc9a378cbff3588e76a5083f5/contracts/ArrakisV2.sol#L138), if the `mintAmount_` is small then liquidity can round down to `0`. In that case, the transaction reverts in `pool.mint()`. It is recommended to move the check at the line [ArrakisV2.sol#L136](https://github.com/ArrakisFinance/v2-core/blob/a9759d1a45bc3a9dc9a378cbff3588e76a5083f5/contracts/ArrakisV2.sol#L136) to line 141.

```solidity
uint128 liquidity = Position.getLiquidityByRange(
    pool,
    me,
    range.lowerTick,
    range.upperTick
);
if (liquidity == 0) continue;

liquidity = SafeCast.toUint128(
    FullMath.mulDiv(liquidity, mintAmount_, ts)
);

pool.mint(me, range.lowerTick, range.upperTick, liquidity, "");
```

2. In the function [`ArrakisV2.rebalance()`](https://github.com/ArrakisFinance/v2-core/blob/a9759d1a45bc3a9dc9a378cbff3588e76a5083f5/contracts/ArrakisV2.sol#L391), there is no check if `rebalanceParams_.mints[i].liquidity == 0`.

```solidity
(uint256 amt0, uint256 amt1) = IUniswapV3Pool(pool).mint(
    address(this),
    rebalanceParams_.mints[i].range.lowerTick,
    rebalanceParams_.mints[i].range.upperTick,
    rebalanceParams_.mints[i].liquidity,
    ""
);
```

3. In the function [`ArrakisV2Resolver.standardRebalance()`](https://github.com/ArrakisFinance/v2-core/blob/a9759d1a45bc3a9dc9a378cbff3588e76a5083f5/contracts/ArrakisV2Resolver.sol#L130), there is no check if `rebalanceParams.mints[i].liquidity == 0`.

```solidity
uint128 liquidity = LiquidityAmounts.getLiquidityForAmounts(
    sqrtPriceX96,
    TickMath.getSqrtRatioAtTick(rangeWeight.range.lowerTick),
    TickMath.getSqrtRatioAtTick(rangeWeight.range.upperTick),
    FullMath.mulDiv(amount0, rangeWeight.weight, hundredPercent),
    FullMath.mulDiv(amount1, rangeWeight.weight, hundredPercent)
);

rebalanceParams.mints[i] = PositionLiquidity({
    liquidity: liquidity,
    range: rangeWeight.range
});
```

### Recommendation

We recommend to move the check for the first case and add checks for cases 2 and 3.

### Status

Fixed at https://github.com/ArrakisFinance/v2-core/blob/37c5957cc25b9143dc333d8f27c66a7c79e60e80/

### Client's comment

> For M-1 issue I only changed in the core and the resolver still could add a liquidity==0 to the mint array of rebalance struct, however this will no longer cause a revert during rebalance since rebalance was altered in the core to simply continue for any mints where liquidity==0.

## First minter can skew the initial ratio

### Description

In the function [`ArrakisV2.mint()`](https://github.com/ArrakisFinance/v2-core/blob/37c5957cc25b9143dc333d8f27c66a7c79e60e80/contracts/ArrakisV2.sol#L84), it is possible to skew the initial ratio in a certain case:

```solidity
// inits
init0M = 1e6
init1M = 1e18
mintAmount = 1

// then because the division is rounded up
amount0 = 1
amount1 = 1
amount0Mint = 1e12
amount1Mint = 1

// the check passes since
min(amount0Mint, amount1Mint) == mintAmount
min(1e12, 1) == 1
```

The minimum `mintAmount` in this case should be `1e12`.

### Recommendation

We recommend to additionally check if `mintAmount * init / denominator == 0`:

```solidity
if (FullMath.mulDiv(mintAmount_, init0M, denominator) == 0) {
    amount0 = 0;
}
if (FullMath.mulDiv(mintAmount_, init1M, denominator) == 0) {
    amount1 = 0;
}
```

### Status

Fixed at https://github.com/ArrakisFinance/v2-core/blob/6cdb18c80a09ecded8b0d5bd4a5d1ee268a03749/

# INFORMATIONAL
