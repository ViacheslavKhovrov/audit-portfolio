# CRITICAL

# HIGH

# MEDIUM

# INFORMATIONAL

## Possible manipulation of reservoirs by directly transferring tokens

### Description

The function [`mintWithReservoir()`](https://github.com/buttonwood-protocol/buttonswap-core/blob/7b4a64319b8232237f7682ef9773ed2dcd94ceb1/src/ButtonswapPair.sol#L506) only specifies the `amountIn` parameter but not the token being used to mint. Let us say the user wants to mint using `token0`. A malicious user can directly transfer `token1` such that the `reservoir1` becomes non zero. Then the contract will use the `token1` approval of `msg.sender` to mint using `token1` instead of `token0`.

### Recommendation

We recommend to ensure such a scenario is not possible in the periphery contracts.

### Status

Acknowledged

### Client's comments 

> These protections already exist in the router contracts.

## `movingAveragePrice0Last` gas optimization

### Description

It is possible to save gas if `movingAveragePrice0Last` is only updated if it is different than `_movingAveragePrice0` at the line [ButtonswapPair.sol#L748](https://github.com/buttonwood-protocol/buttonswap-core/blob/7b4a64319b8232237f7682ef9773ed2dcd94ceb1/src/ButtonswapPair.sol#L748).

### Recommendation

We recommend to update `movingAveragePrice0Last` only if it changes.

### Status

Acknowledged

### Client's comments

> The proposed change adds a warm SLOAD from movingAveragePrice0Last every swap, costing 100 gas. It can at times save a warm same-value SSTORE to movingAveragePrice0Last on some swaps, saving 100 gas. Due to movingAveragePrice0 being calculated by interpolating values based on current time, it is very often going to be different from the last stored value (the most likely exception is when two swaps occur in the same block).
> On balance we believe always updating it results in lower average gas consumption rather than only updating when the value has changed.

## No sanity checks in setter functions

### Description

### Recommendation

We recommend to 

### Status

DOUBLE
