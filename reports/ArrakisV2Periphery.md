# CRITICAL

# HIGH

## DoS of `RouterSwapExecutor` with non zero approvals

### Description

In the function [`RouterSwapExecutor.swap()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/RouterSwapExecutor.sol#L52), `swapRouter` and `swapPayload` are user chosen.

```solidity
if (swapAndAddData_.swapData.zeroForOne) {
    balanceBefore = token0.balanceOf(address(this));
    token0.safeApprove(
        swapAndAddData_.swapData.swapRouter,
        swapAndAddData_.swapData.amountInSwap
    );
} else {
    balanceBefore = token1.balanceOf(address(this));
    token1.safeApprove(
        swapAndAddData_.swapData.swapRouter,
        swapAndAddData_.swapData.amountInSwap
    );
}
(bool success, ) = swapAndAddData_.swapData.swapRouter.call(
    swapAndAddData_.swapData.swapPayload
);
```

An attacker can choose `swapRouter` as any token different than `token0` and `token1` and approve non zero value for routers. Since the token chosen by the attacker is different than `token0` and `token1`, its allowance will not be reset to `0`.
Then when a user tries to swap `safeApprove()` will revert since there is already a non zero approval.

```solidity
function safeApprove(
    IERC20 token,
    address spender,
    uint256 value
) internal {
    // safeApprove should only be called when setting an initial allowance,
    // or when resetting it to zero. To increase and decrease it, use
    // 'safeIncreaseAllowance' and 'safeDecreaseAllowance'
    require(
        (value == 0) || (token.allowance(address(this), spender) == 0),
        "SafeERC20: approve from non-zero to non-zero allowance"
    );
    _callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, value));
}
```

This way an attacker can DoS every call to `swapAndAddLiquidity()` by frontrunning.

### Recommendation

We recommend to use only whitelisted routers as `swapRouter`.

### Status

Fixed at https://github.com/ArrakisFinance/v2-periphery/blob/3eaf254bfd540c1ebfd75c2ced3ca914c0488ede

# MEDIUM

## Growth factor calculation includes manager fees

### Description

When calculating `totalUnderlyingWithFees`, the `Underlying` library subtracts admin fees from `underlying.fee0` and `underlying.fee1` [Underlying.sol#L476-L493](https://github.com/ArrakisFinance/v2-core/blob/a9759d1a45bc3a9dc9a378cbff3588e76a5083f5/contracts/libraries/Underlying.sol#L476-L493):

```solidity
(uint256 fee0After, uint256 fee1After) = subtractAdminFees(
    fee0,
    fee1,
    arrakisV2.managerFeeBPS()
);

amount0 +=
    fee0After +
    IERC20(underlyingPayload_.token0).balanceOf(
        underlyingPayload_.self
    ) -
    arrakisV2.managerBalance0();
amount1 +=
    fee1After +
    IERC20(underlyingPayload_.token1).balanceOf(
        underlyingPayload_.self
    ) -
    arrakisV2.managerBalance1();
```

Thus,

```solidity
underlying.amount0 = amount0 + underlying.leftOver0 + underlying.fee0 - managerFee0
```

But this is not done in the function [`ArrakisV2StaticManager.compoundFees()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/ArrakisV2StaticManager.sol#L70-L90), `underlying.fee0` and `underlying.fee1` include fees of the manager.

```solidity
uint256 liquidity0 = underlying.amount0 -
    (underlying.leftOver0 + underlying.fee0);
uint256 liquidity1 = underlying.amount1 -
    (underlying.leftOver1 + underlying.fee1);
uint256 proportion0 = liquidity0 > 0
    ? FullMath.mulDiv(
        underlying.leftOver0 + underlying.fee0,
        hundredPercent,
        liquidity0
    )
    : type(uint256).max;
```

Which means `proportion0`, `proportion1` and consequently `growthFactor` will be higher than they are supposed to be. When the rebalance withdraws tokens from the pool to the vault, it will lock manager fees and this could lead to mints reverting due to not having enough tokens.

### Recommendation

We recommend to subtract manager fees from `underlying.fee0` and `underlying.fee1`.

### Status

Fixed at https://github.com/ArrakisFinance/v2-periphery/blob/3eaf254bfd540c1ebfd75c2ced3ca914c0488ede

# INFORMATIONAL

## No address zero checks

### Description

In the contracts there are several instances where the address is not checked for address zero. For example, at the line [ArrakisV2GaugeFactory.sol#L31](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/ArrakisV2GaugeFactory.sol#L31), the `stakingToken_` can be `address(0)`.
Other instances: [ArrakisV2StaticDeployer.sol#L58-L61](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/ArrakisV2StaticDeployer.sol#L58-L61), [RouterSwapExecutor.sol#L27](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/RouterSwapExecutor.sol#L27), [RouterSwapResolver.sol#L26-L27](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/RouterSwapResolver.sol#L26-L27), [ArrakisV2GaugeFactoryStorage.sol#L32](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/abstract/ArrakisV2GaugeFactoryStorage.sol#L32), [ArrakisV2RouterStorage.sol#L43-45](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/abstract/ArrakisV2RouterStorage.sol#L43-45), [ArrakisV2StaticManagerStorage.sol#L27-28](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/abstract/ArrakisV2StaticManagerStorage.sol#L27-28).

### Recommendation

We recommend to add zero address checks.

### Status

DOUBLE

## No check if gauge is in `_gauges`

### Description

The functions [`ArrakisV2GaugeFactory.addGaugeReward()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/ArrakisV2GaugeFactory.sol#L44), [`ArrakisV2GaugeFactory.setGaugeRewardDistributor()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/ArrakisV2GaugeFactory.sol#L61) take parameter `gauge_`, but there is no check if `gauge_` is in the address set `_gauges`. The owner can use these functions for gauges not deployed by the factory.

### Recommendation

We recommend to check if `gauge_` is in the address set `_gauges`.

### Status

DOUBLE

## Lack of events

### Description

There is a lack of events in some contracts when the state changes:

1. The functions [`ArrakisV2GaugeFactory.addGaugeReward()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/ArrakisV2GaugeFactory.sol#L44), [`ArrakisV2GaugeFactory.setGaugeRewardDistributor()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/ArrakisV2GaugeFactory.sol#L61) do not emit an event, neither does the gauge contract. It could be useful to track this data. 
2. Since anyone can call the function [`ArrakisV2StaticManager.setStaticVault()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/ArrakisV2StaticManager.sol#L32), it should emit an event to track which vaults are added.
3. The function [`ArrakisV2StaticManager.withdrawAndCollectFees()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/ArrakisV2StaticManager.sol#L125) doesn't emit an event with which vaults were withdrawn from and which tokens were transferred.
4. The function [`ArrakisV2RouterStorage.updateSwapExecutor()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/abstract/ArrakisV2RouterStorage.sol#L66) doesn't emit an event when the swap executor changes.

### Recommendation

We recommend to add events where necessary.

### Status

Acknowledged

## Low-level call doesn't forward the revert reason

### Description

At the line [RouterSwapExecutor.sol#L55](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/RouterSwapExecutor.sol#L55):

```solidity
(bool success, ) = swapAndAddData_.swapData.swapRouter.call(
    swapAndAddData_.swapData.swapPayload
);
require(success, "swap: low-level call failed");
```

The low-level call doesn't forward the revert reason. If the transaction reverts, it can be useful for a user to know why it reverted. 

### Recommendation

We recommend to forward the revert reason.

### Status

Acknowledged

## Insufficient address zero checks

### Description

At the line [ArrakisV2GaugeFactoryStorage.sol#L43-L49](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/abstract/ArrakisV2GaugeFactoryStorage.sol#L43-L49), the `initialize()` parameters are checked to be non zero address. However, it is possible to send one non zero address and others with `address(0)`. The check passes since logical `OR` is used instead of `AND`.

The second instance at the line [ArrakisV2GaugeFactoryStorage.sol#L63-L68](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/abstract/ArrakisV2GaugeFactoryStorage.sol#L63-L68).

### Recommendation

We recommend to replace `||` with `&&`.

### Status

DOUBLE

## Owner can renounce ownership

### Description

The `OwnableUpgradeable` contract implements a `renounceOwnership()` function which will remove any functionality that is only available to the owner. The contracts `ArrakisV2GaugeFactory`, `ArrakisV2RouterStorage`, `ArrakisV2StaticManagerStorage` inherit from `OwnableUpgradeable`, but do not override the `renounceOwnership()` function. The contracts have functions that are callable only by the owner, so if `renounceOwnership()` is called by mistake, these functions will be uncallable.

### Recommendation

We recommend to override `renounceOwnership()` to revert on call.

### Status

Acknowledged

## `ArrakisV2RouterStorage` allows direct ETH transfers

### Description

`ArrakisV2RouterStorage` has the [`receive()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/abstract/ArrakisV2RouterStorage.sol#L48) fallback to receive ETH sent by the `WETH` contract. Currently, any user can send ETH directly. It would be better to allow only the `WETH` contract to send ETH. It is possible to force ETH by using `selfdestruct`, but this would disallow locking ETH sent by mistake.

### Recommendation

We recommend to change to:

```solidity
receive() external payable {
    require(msg.sender == weth, 'Not WETH');
}
```

### Status

Acknowledged

## Attacker can steal tokens and allowance from `RouterSwapExecutor`

### Description

In the function [`RouterSwapExecutor.swap()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/RouterSwapExecutor.sol#L52), `swapRouter` and `swapPayload` are user chosen. An attacker can choose `swapRouter` as any token and transfer from `RouterSwapExecutor` balance or allowance. The `RouterSwapExecutor` should not have any tokens and allowance, but this is not an intended use of `swap()` function.

### Recommendation

We recommend to use only whitelisted routers as `swapRouter`.

### Status

Acknowledged

## `RouterSwapExecutor` transfers all balance after swapping

### Description

In the function [`RouterSwapExecutor.swap()`](https://github.com/ArrakisFinance/v2-periphery/blob/cab630396506aad825d838f98d60d287ed49c0b9/contracts/RouterSwapExecutor.sol#L68), the `amount1Diff` is calculated as `balance1`. If `RouterSwapExecutor` has non zero `token1` balance before the swap, it will be sent to the router and the leftover to the user. The user will intentionally or unintentionally receive the tokens from the contract.

### Recommendation

We recommend to calculate `amount1Diff` the same way as `amount0Diff`. If the locking of tokens in `RouterSwapExecutor` is undesired, add a recover method to retrieve tokens.

### Status

Acknowledged

## Permitted tokens might not match vault tokens

### Description

In the `ArrakisV2Router` the methods that use `Permit2` do not check if the permitted tokens match the vault tokens. If the `ArrakisV2Router` has balance of vault tokens, it will pull different tokens and use tokens from its own balance to add liquidity.

### Recommendation

We recommend to check if the permitted tokens match vault tokens.

### Status

DOUBLE
