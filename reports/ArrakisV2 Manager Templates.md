# CRITICAL

# HIGH

# MEDIUM

# INFORMATIONAL

## No address zero checks

### Description

There are no address zero checks at the lines [contracts/SimpleManager.sol#L74](https://github.com/ArrakisFinance/v2-manager-templates/blob/1d507a2beaa9c0e785bac7dd943c77964fedaef3/contracts/SimpleManager.sol#L74), [contracts/SimpleManager.sol#L78](https://github.com/ArrakisFinance/v2-manager-templates/blob/1d507a2beaa9c0e785bac7dd943c77964fedaef3/contracts/SimpleManager.sol#L78).

### Recommendation

We recommend to add address zero checks.

### Status

DOUBLE

## No way to remove vaults and modify vault info

### Description

The function [`SimpleManager.initManagement()`](https://github.com/ArrakisFinance/v2-manager-templates/blob/1d507a2beaa9c0e785bac7dd943c77964fedaef3/contracts/SimpleManager.sol#L84) adds a vault to `vaults` and sets its info. However, there is no way to undo this action or modify vault info after `initManagement()`. We recommend to add a function to remove a vault from `vaults` that is callable by the `SimpleManager` owner. One needs to consider the possibility of vault owners frontrunning the `rebalance()` function. This way, modifying vault info would require the `SimpleManager` owner to remove the vault and the vault owner to add the vault again with different parameters.

### Recommendation

We recommend to add a function to remove a vault from `vaults`.

### Status

Acknowledged

## Max deviation is not bounded

### Description

In the function [`SimpleManager.initManagement()`](https://github.com/ArrakisFinance/v2-manager-templates/blob/1d507a2beaa9c0e785bac7dd943c77964fedaef3/contracts/SimpleManager.sol#L84), the parameter `params.maxDeviation` is not bounded from above. If the vault owner sets the wrong parameter, an operator can sandwich the rebalance by swapping outside of the rebalance.

### Recommendation

We recommend to add an upper limit for `params.maxDeviation`.

### Status

Acknowledged

## No checks if vault is in `vaults`

### Description

1. In the function [`SimpleManager.rebalance()`](https://github.com/ArrakisFinance/v2-manager-templates/blob/1d507a2beaa9c0e785bac7dd943c77964fedaef3/contracts/SimpleManager.sol#L113), an operator can burn liquidity from a vault in which `SimpleManager` is the manager, but it `initManagement()` was not called yet.

2. In the function [`SimpleManager.withdrawAndCollectFees()`](https://github.com/ArrakisFinance/v2-manager-templates/blob/1d507a2beaa9c0e785bac7dd943c77964fedaef3/contracts/SimpleManager.sol#L203), the owner can withdraw manager balance from a vault in which `SimpleManager` is the manager, but it `initManagement()` was not called yet.

### Recommendation

We recommend adding checks if vault is in `vaults`.

### Status

Fixed at https://github.com/ArrakisFinance/v2-manager-templates/commit/74717eb9656b03e13385c701694b62a35a03d26b

## Unused function `_includesAddress()`

### Description

The function [`SimpleManager._includesAddress()`](https://github.com/ArrakisFinance/v2-manager-templates/blob/1d507a2beaa9c0e785bac7dd943c77964fedaef3/contracts/SimpleManager.sol#L357) is unused and should be removed.

### Recommendation

We recommend removing this function.

### Status

DOUBLE

## No checks for stale data

### Description

There are no checks if the data received from the price feed is fresh when calling `latestRoundData()` at the lines [contracts/oracles/ChainLinkOracle.sol#L35](https://github.com/ArrakisFinance/v2-manager-templates/blob/1d507a2beaa9c0e785bac7dd943c77964fedaef3/contracts/oracles/ChainLinkOracle.sol#L35), [contracts/oracles/ChainLinkOracle.sol#L60](https://github.com/ArrakisFinance/v2-manager-templates/blob/1d507a2beaa9c0e785bac7dd943c77964fedaef3/contracts/oracles/ChainLinkOracle.sol#L60).

### Recommendation

We recommend checking if the return value `updatedAt` of `latestRoundData()` call is recent.

### Status

DOUBLE
