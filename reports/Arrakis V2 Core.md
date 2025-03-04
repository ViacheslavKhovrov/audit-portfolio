# CRITICAL

Not found

# HIGH

## Any user can burn all liquidity from the pools to the vault

### Description

In the function `burn()` at the lines [ArrakisV2.sol#L172-L183](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L172-L183):
```solidity
Withdraw memory withdraw = _withdraw(
    IUniswapV3Pool(
        factory.getPool(
            address(token0),
            address(token1),
            burns_[i].range.feeTier
        )
    ),
    burns_[i].range.lowerTick,
    burns_[i].range.upperTick,
    burns_[i].liquidity
);
```

Any user with LP tokens can use `burn()` function to burn all of the liquidity from the pools since they can choose the parameters `burns_[i].liquidity` to equal all of the liquidity of the range. The Uniswap pools will have less liquidity as a result.

### Recommendation

It is recommended to compute the liquidity to burn proportionally to `burnAmount_` directly in the function.

### Status
DOUBLE

## First minter can change LP token pricing

### Description

At the lines [ArrakisV2.sol#L83-L84](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L83-L84):
```solidity
amount0 = FullMath.mulDivRoundingUp(mintAmount_, current0, denominator);
amount1 = FullMath.mulDivRoundingUp(mintAmount_, current1, denominator);
```

When `totalSupply` is `0`, the variables `current0, current1, denominator` are set to `init0, init1, 1 ether` respectively. If `init0` and `init1` are less than `1e18`, then the first minter can mint `mintAmount_ = 1 wei` for `amount0 = 1 wei` and `amount1 = 1 wei` regardless of the values `init0` and `init1`.

### Recommendation

It is recommended to ensure `amount0, amount1` are proportional to `init0, init1`.

### Status
Fixed at https://github.com/ArrakisFinance/v2-core/blob/23ac7829fe8b84968e1319c483e8c374ca785127/contracts/ArrakisV2.sol#L97

## Vault can renew the term for free

### Description

The vault owner mints `1 wei` when deploying a vault with `openTerm()`. Then they can renew term for free since in the function `renewTerm()` at the line [PALMTerms.sol#L153](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/PALMTerms.sol#L153), `emolumentShares` are calculated from the balance of `PALMTerms` and the function `increaseLiquidity()` doesn't mint new LP tokens.

### Recommendation

It is recommended to mint new LP tokens in `increaseLiquidity()`.

### Status
Fixed at https://github.com/ArrakisFinance/v2-palm/blob/46546698096f0f4cd86381cd06be28d43a50a2ea/contracts/PALMTerms.sol#L66

## Vault owner can lose tokens when increasing liquidity

### Description

At the line [PALMTerms.sol#L170](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/PALMTerms.sol#L170). If the vault owner burns all of the LP tokens owned by `PALMTerms` with `decreaseLiquidity()`, then any future call to `increaseLiquidity()` would lock the tokens in the vault's address.

### Recommendation

It is recommended to mint new LP tokens in `increaseLiquidity()`.

### Status
DOUBLE

# MEDIUM

## Manager can set arbitrary `managerFeeBPS`

### Description

In the function `_applyFees()` at the line [ArrakisV2.sol#L461](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L461):
```solidity
uint16 managerFeeBPS = Manager.getManagerFeeBPS(manager);
```

The manager can return any `managerFeeBPS`. If `managerFeeBPS + arrakisFeeBPS` is bigger than `10000`, then admin balances can be bigger than balances of vault tokens in the contract.

### Recommendation

It is recommended to check if `managerFeeBPS + arrakisFeeBPS <= 10000`.

### Status
DOUBLE

## No check if `liquidity > 0` in `standardBurnParams()`

### Description

In the function `standardBurnParams()` at the lines [ArrakisV2Resolver.sol#L220-L225](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Resolver.sol#L220-L225):
```solidity
burns[i] = BurnLiquidity({
    liquidity: SafeCast.toUint128(
        FullMath.mulDiv(liquidity, amountToBurn_, totalSupply)
    ),
    range: ranges[i]
});
```

There is no check if `burns[i].liquidity > 0`. If `burns[i].liquidity = 0`, then `burn()` in `ArrakisV2` will revert.

### Recommendation

It is recommended to check if `burns[i].liquidity > 0` and not include range if `liquidity` = 0.

### Status
Fixed at https://github.com/ArrakisFinance/v2-core/blob/50706e5a9ebdfa290a0823c24c745b85398b6452

## Function `_removeOperators()` doesn't pop element from array `operators` 

### Description

At the lines [PALMManagerStorage.sol#L404-L413](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMManagerStorage.sol#L404-L413):
```solidity
function _removeOperators(address[] memory operators_) internal {
    for (uint256 i = 0; i < operators_.length; i++) {
        (bool isOperator, uint256 index) = _isOperator(operators_[i]);
        require(isOperator, "PALMManager: no operator");

        delete operators[index];
    }

    emit RemoveOperators(address(this), operators_);
}
``` 

The element `operators[index]` is zeroed, but the length of the `operators` array doesn't decrease.

### Recommendation

```solidity
(bool isOperator, uint256 index) = _isOperator(operators_[i]);
require(isOperator, "PALMManager: no operator");

operators[index] = operators[operators.length - 1];
operators.pop();
``` 

### Status
DOUBLE

## Anyone can call `renewTerm()`

### Description

In the function `renewTerm()` at the line [PALMTerms.sol#L143](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/PALMTerms.sol#L143), there is no check if the caller is the owner of the vault. An attacker can frontrun `closeTerm()` and call `renewTerm()`, then the vault would pay the emolument twice.

### Recommendation

It is recommended to add the modifier `requireIsOwner()`.

### Status
Acknowledged

### Client's comments
> By design. RenewTerm can be called only when term management time ends. If client want to terminate the term with palm, they can call closeTerm function before end of management time. If they do not, renewal is automatic and we (or anyone) will promptly call renewTerm to extract the fee of the last epoch.

# INFORMATIONAL

## No max limit for `ranges`

### Description

In the function `rebalance()` at the line [ArrakisV2.sol#L241](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L241).

The manager can add ranges, but there is no max limit for the number of ranges. If the `ranges` array is too big, it will be impossible to mint and burn LP tokens since there will not be enough gas in the block.

### Recommendation

It is recommended to limit the maximum number of ranges. 

### Status
Acknowledged

## Unknown recipient in swap payload

### Description

In the function `_rebalance()` at the lines [ArrakisV2.sol#L368-L371](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L368-L371):
```solidity
(bool success, ) = rebalanceParams_.swap.router.call(
    rebalanceParams_.swap.payload
);
require(success, "SC");
```

In the `rebalanceParams_.swap.payload` the manager can put themselves instead of `address(this)` as recipient.

### Recommendation

It is recommended to add checks for swap payload.

### Status
DOUBLE

## Fee amount sent to Gelato is not limited

### Description

In the function `_preExec()` at the line [PALMManager.sol#L57](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/PALMManager.sol#L57):
```solidity
Address.sendValue(gelatoFeeCollector, feeAmount_);
```

The fee amount sent from the vault's balance during the rebalance is not limited. If an operator is compromised, they can drain vault's balance to Gelato.

### Recommendation

It is recommended to add a max limit on the amount sent.

### Status
DOUBLE

## Unused imports

### Description

- At the lines [ArrakisV2.sol#L19-L20](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L19-L20), the imports `ERC20` and `SafeCast` are not used.

- At the line [ArrakisV2Factory.sol#L14](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Factory.sol#L14), the import `ERC1967Upgrade` is not used.

- At the lines [ArrakisV2FactoryStorage.sol#L12-L14](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/abstract/ArrakisV2FactoryStorage.sol#L12-L14), the import `Initializable` is not used.

- At the line [Manager.sol#L5](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/libraries/Manager.sol#L5), the import `Range` is not used.

- At the lines [Pool.sol#L7-L9](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/libraries/Pool.sol#L7-L9), the import `IUniswapV3Pool` is a duplicate.

- At the line [ArrakisV2Resolver.sol#L16](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Resolver.sol#L16), the import `ERC20` is not used.

- At the line [ArrakisV2Resolver.sol#L32](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Resolver.sol#L32), the import `SwapPayload` is not used.

- At the lines [PALMTermsStorage.sol#L16-L18](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMTermsStorage.sol#L16-L18), the import `ReentrancyGuardUpgradeable` is not used.

### Recommendation

It is recommended to remove unused imports.

### Status
DOUBLE

## Variable `totalSupply` shadows function

### Description

At the lines [ArrakisV2.sol#L63](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L63), [ArrakisV2.sol#L106](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L106):
```solidity
uint256 totalSupply = totalSupply();
```

The variable `totalSupply` shadows function `totalSupply()`.

### Recommendation

It is recommended to rename the variable.

### Status
Fixed at https://github.com/ArrakisFinance/v2-core/blob/23ac7829fe8b84968e1319c483e8c374ca785127

## Possible to use LP token as vault token

### Description

In the function `mint()` at the lines [ArrakisV2.sol#L86-L94](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L86-L94):
```solidity
_mint(receiver_, mintAmount_);

// transfer amounts owed to contract
if (amount0 > 0) {
    token0.safeTransferFrom(msg.sender, me, amount0);
}
if (amount1 > 0) {
    token1.safeTransferFrom(msg.sender, me, amount1);
}
```

The LP token is minted before `token0` and `token1` are pulled to the contract, which means that it is possible to use LP token of the vault as `token0` or `token1`.

### Recommendation

It is recommended to mint LP tokens after pulling `token0` and `token1`.

### Status
Acknowledged

## Possible to burn zero LP tokens

### Description

In the function `burn()` at the line [ArrakisV2.sol#L101](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L101), the parameter `burnAmount_` is not checked to be bigger than `0`.

### Recommendation

It is recommended to check if `burnAmount_ > 0`.

### Status
Fixed at https://github.com/ArrakisFinance/v2-core/blob/23ac7829fe8b84968e1319c483e8c374ca785127

## Mint and burn `receiver_` can be address zero

### Description

In the function `mint()` at the line [ArrakisV2.sol#L52](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L52), the parameter `receiver_` is not checked for `address(0)`.

In the function `burn()` at the line [ArrakisV2.sol#L101](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L101), the parameter `receiver_` is not checked for `address(0)`.

### Recommendation

It is recommended to check if `receiver_ != address(0)`.

### Status
DOUBLE

## No check if range exists

### Description

In the function `burn()` at the lines [ArrakisV2.sol#L172-L183](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L172-L183):
```solidity
Withdraw memory withdraw = _withdraw(
    IUniswapV3Pool(
        factory.getPool(
            address(token0),
            address(token1),
            burns_[i].range.feeTier
        )
    ),
    burns_[i].range.lowerTick,
    burns_[i].range.upperTick,
    burns_[i].liquidity
);
```

The parameters `burns_[i].range` are not checked if they exist.

In the function `_rebalance()` at the lines [ArrakisV2.sol#L330-L335](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L330-L335):
```solidity
Withdraw memory withdraw = _withdraw(
    pool,
    rebalanceParams_.removes[i].range.lowerTick,
    rebalanceParams_.removes[i].range.upperTick,
    rebalanceParams_.removes[i].liquidity
);
```

The parameters `rebalanceParams_.removes[i].range` are not checked if they exist.

### Recommendation

It is recommended to check if the ranges exist.

### Status
DOUBLE

## Incorrect event emit

### Description

In the function `burn()` at the line [ArrakisV2.sol#L216](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L216):
```solidity
emit LogUncollectedFees(underlying.fee0, underlying.fee1);
```

Some fees will be collected when burning liquidity, so this event emit `LogUncollectedFees()` is incorrect.

### Recommendation

It is recommended to emit `LogUncollectedFees()` before the `return` statement at the line [ArrakisV2.sol#L159](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L159).

### Status
Acknowledged

## `TODO` comments

### Description

At the lines:
- [ArrakisV2.sol#L238](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L238)
- [ArrakisV2Resolver.sol#L111](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Resolver.sol#L111)

`TODO` comments should be removed before deployment.

### Recommendation

It is recommended to remove `TODO` comments.

### Status
Fixed at https://github.com/ArrakisFinance/v2-core/blob/23ac7829fe8b84968e1319c483e8c374ca785127

## Gas expensive removal of elements from array `ranges`

### Description

In the function `rebalance()` at the lines [ArrakisV2.sol#L269-L274](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L269-L274):
```solidity
delete ranges[index];

for (uint256 j = index; j < ranges.length - 1; j++) {
    ranges[j] = ranges[j + 1];
}
ranges.pop();
```

When removing ranges, reading and writing to storage in a loop consumes a lot of gas. It would be much more gas efficient to assign and pop to remove an element. The order of ranges is not important anywhere in the contract.

### Recommendation

```solidity
ranges[index] = ranges[ranges.length - 1];
ranges.pop();
```

### Status
DOUBLE

## Using a number literal

### Description

- In the function `_applyFees()` at the lines [ArrakisV2.sol#L462-L45](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L462-L465)
- At the lines [ArrakisV2Resolver.sol#L133-L134](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Resolver.sol#L133-L134)
- At the line [ArrakisV2Resolver.sol#L292](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Resolver.sol#L292)
- At the line [PALMTermsStorage.sol#L87](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMTermsStorage.sol#L87)

The literal `10000` can be replaced with a constant variable for better readability.

### Recommendation

It is recommended to use a constant variable instead of a literal.

### Status
Acknowledged

## Inadequate view functions in `ArrakisV2Storage`

### Description

At the lines [ArrakisV2Storage.sol#L65-L66](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/abstract/ArrakisV2Storage.sol#L65-L66):
```solidity
EnumerableSet.AddressSet internal _pools;
EnumerableSet.AddressSet internal _routers;
```

The are no view functions for pools and routers used in the vault.

### Recommendation

It is recommended to add view functions for pools and routers.

### Status
Acknowledged

## Possible to add `address(0)` as router

### Description

In the function `_whitelistRouters()` at the lines [ArrakisV2Storage.sol#L241-L252](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/abstract/ArrakisV2Storage.sol#L241-L252), there is no check if `routers_[i]` is not `address(0)`.

Low-level call to `address(0)` will return `success = true` at the lines [ArrakisV2.sol#L368-L371](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2.sol#L368-L371):
```solidity
(bool success, ) = rebalanceParams_.swap.router.call(
    rebalanceParams_.swap.payload
);
require(success, "SC");
```

### Recommendation

It is recommended to check if router is not `address(0)`.

### Status
DOUBLE

## Possible to set `address(0)` as manager

### Description

In the function `setManager()` at the lines [ArrakisV2Storage.sol#L192-L194](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/abstract/ArrakisV2Storage.sol#L192-L194), there is no check if `manager_` is not `address(0)`.

### Recommendation

It is recommended to check if `manager_` is not `address(0)`.

### Status
DOUBLE

## Function `removePools()` doesn't remove ranges

### Description

In the function `removePools()` at the lines [ArrakisV2Storage.sol#L169-L176](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/abstract/ArrakisV2Storage.sol#L169-L176):

```solidity
function removePools(address[] calldata pools_) external onlyOwner {
    for (uint256 i = 0; i < pools_.length; i++) {
        require(_pools.contains(pools_[i]), "NP");

        _pools.remove(pools_[i]);
    }
    emit LogRemovePools(pools_);
}
```

The pool is only removed from `_pools`, but there might be active positions with same fee tier in `_ranges`.

### Recommendation

It is recommended to remove ranges with the same fee tier when removing a pool and ensure they have no liquidity.

### Status
Acknowledged

## Functions can be declared as `external`

### Description

- The function `vaults()` at the line [ArrakisV2Factory.sol#L57](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Factory.sol#L57) can be `external` since it is not used internally.

- The function `getProxyAdmin()` at the line [ArrakisV2FactoryStorage.sol#L86](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/abstract/ArrakisV2FactoryStorage.sol#L86) can be `external` since it is not used internally.

- The function `getProxyImplementation()` at the line [ArrakisV2FactoryStorage.sol#L96](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/abstract/ArrakisV2FactoryStorage.sol#L96) can be `external` since it is not used internally.

- The function `getAmountsForLiquidity()` at the line [ArrakisV2Resolver.sol#L266](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Resolver.sol#L266) can be `external` since it is not used internally.

### Recommendation

It is recommended to change these functions to `external`.

### Status
Fixed at https://github.com/ArrakisFinance/v2-core/blob/23ac7829fe8b84968e1319c483e8c374ca785127

## Function `vaults()` may run out of gas

### Description

The function `vaults()` at the lines [ArrakisV2Factory.sol#L57-L65](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Factory.sol#L57-L65):
```solidity
function vaults() public view returns (address[] memory) {
    uint256 length = numVaults();
    address[] memory vs = new address[](length);
    for (uint256 i = 0; i < length; i++) {
        vs[i] = _vaults.at(i);
    }

    return vs;
}
```

If there are a lot of vaults deployed, this function will be unusable since there will not be enough gas in the block to loop over all of the vaults.

### Recommendation

It is recommended to change the function to get one vault with parameter `index`.

### Status
Fixed at https://github.com/ArrakisFinance/v2-core/blob/23ac7829fe8b84968e1319c483e8c374ca785127

## Unnecessary external calls

### Description

At the lines:
- [ArrakisV2Helper.sol#L36](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Helper.sol#L36)
- [ArrakisV2Helper.sol#L71](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Helper.sol#L71)
- [ArrakisV2Helper.sol#L88](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Helper.sol#L88)
- [ArrakisV2Resolver.sol#L78](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Resolver.sol#L78)
- [ArrakisV2Resolver.sol#L122](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Resolver.sol#L122)
- [ArrakisV2Resolver.sol#L206](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Resolver.sol#L206)

The external calls `vault_.factory()` are unnecessary since `factory` is already available as an immutable variable. 

### Recommendation

It is recommended to use variable `factory` instead of an external call.

### Status
Fixed at https://github.com/ArrakisFinance/v2-core/blob/23ac7829fe8b84968e1319c483e8c374ca785127

## Inefficient function `ranges()` in `ArrakisV2Helper`

### Description

At the line [ArrakisV2Helper.sol#L171](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Helper.sol#L171), the function `ranges()` is gas inefficient. It can be replaced with a getter function in `ArrakisV2Storage` that would return `ranges`.

### Recommendation

It is recommended to add a getter function in `ArrakisV2Storage` to return `ranges`.

### Status
Fixed at https://github.com/ArrakisFinance/v2-core/blob/23ac7829fe8b84968e1319c483e8c374ca785127

## Unused variable `swapRouter` in `ArrakisV2Resolver`

### Description

At the line [ArrakisV2Resolver.sol#L39](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/ArrakisV2Resolver.sol#L39)

The variable `swapRouter` is unused.

### Recommendation

It is recommended to remove the unused variable.

### Status
DOUBLE

## Duplicate code

### Description

At the lines [ArrakisV2Resolver.sol#L171-L178](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/ArrakisV2Resolver.sol#L171-L178), the computation of leftovers is unnecessary. It is possible to use `totalUnderlyingWithFeesAndLeftOver()` from `ArrakisV2Helper`.

### Recommendation

It is recommended to use `totalUnderlyingWithFeesAndLeftOver()`.

### Status
DOUBLE

## Array `operators` can be changed to `EnumerableSet.AddressSet`

### Description

At the line [PALMManagerStorage.sol#L65](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMManagerStorage.sol#L65). 

The array `operators` has costly operations when adding, removing operators and checking if an operator exists. It can be optimized if `operators` is a `EnumerableSet.AddressSet`.

### Recommendation

It is recommended to change `operators` to a `EnumerableSet.AddressSet`.

### Status
Fixed at https://github.com/ArrakisFinance/v2-palm/blob/90e5c17e6c01dcf7158ee4aab82b133bff198726

## `gelatoFeeCollector` can be set to `address(0)`

### Description

At the lines [PALMManagerStorage.sol#L138](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMManagerStorage.sol#L138), [PALMManagerStorage.sol#L217](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMManagerStorage.sol#L217).

The variable `gelatoFeeCollector` can be set to `address(0)`.

### Recommendation

It is recommended to ensure `gelatoFeeCollector` can never be `address(0)`.

### Status
Fixed at https://github.com/ArrakisFinance/v2-palm/blob/90e5c17e6c01dcf7158ee4aab82b133bff198726

## Possible to add vaults directly to `PALMManager`

### Description

At the line [PALMManagerStorage.sol#L166](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMManagerStorage.sol#L166), [PALMManagerStorage.sol#L82-L88](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMManagerStorage.sol#L82-L88).
```solidity
modifier onlyPALMTermsVaults(address vault) {
    require(
        IArrakisV2(vault).owner() == terms,
        "PALMManager: owner no PALMTerms"
    );
    _;
}
```

Users can their own vaults to `PALMManager` if they modify function `IArrakisV2(vault).owner()` to return `terms`. Then these vaults would be managed and available for rebalance.

### Recommendation

It is recommended to ensure that only `PALMTerms` can add vaults to `PALMManager`.

### Status
DOUBLE

## Duplicate code

### Description

At the line [PALMManagerStorage.sol#L179](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMManagerStorage.sol#L179):
```solidity
require(vaults[vault_].termEnd != 0, "PALMManager: Vault not managed");
```

This check is done in the modifier `onlyManagedVaults()` at the lines [PALMManagerStorage.sol#L90-L93](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMManagerStorage.sol#L90-L93).

### Recommendation

It is recommended to use the modifier `onlyManagedVaults()`.

### Status
DOUBLE

## Can fund vault balance with `msg.value = 0`

### Description

In the function `fundVaultBalance()` at the lines [PALMManagerStorage.sol#L275-L283](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMManagerStorage.sol#L275-L283), there is no check if `msg.value > 0`.

### Recommendation

It is recommended to check if `msg.value > 0`.

### Status
Fixed at https://github.com/ArrakisFinance/v2-palm/blob/90e5c17e6c01dcf7158ee4aab82b133bff198726

## Unnecessary assignment

### Description

At the line [PALMManagerStorage.sol#L371](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMManagerStorage.sol#L371):
```solidity
vaults[vault_].balance = 0;
```

The assignment to `0` is not necessary since `delete vaults[vault_];` zeroes `vaults[vault_].balance`.

### Recommendation

It is recommended to remove the assignment `vaults[vault_].balance = 0;`

### Status
DOUBLE

## Irrelevant comments

### Description

At the lines [PALMTerms.sol#L83](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/PALMTerms.sol#L83), [PALMTerms.sol#L92](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/PALMTerms.sol#L92), the comments are irrelevant.

### Recommendation

It is recommended to move the comment at line [PALMTerms.sol#L83](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/PALMTerms.sol#L83) and remove the comment at line [PALMTerms.sol#L92](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/PALMTerms.sol#L92).

### Status
Fixed at https://github.com/ArrakisFinance/v2-palm/blob/90e5c17e6c01dcf7158ee4aab82b133bff198726

## User can mint LP tokens to themselves in `PALMTerms`

### Description

At the lines [PALMTerms.sol#L93-L102](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/PALMTerms.sol#L93-L102), `token0` and `token1` are transferred to the contract before `vaultV2.setRestrictedMint(address(this));` is called. If `token0` or `token1` is an `ERC777` token, then the caller can mint LP tokens before the mint is restricted.

### Recommendation

It is recommended to add restricted mint as a vault deployment parameter.

### Status
Fixed at https://github.com/ArrakisFinance/v2-palm/blob/90e5c17e6c01dcf7158ee4aab82b133bff198726

## `setRestrictedMint()` can be frontrun

### Description

At the lines [ArrakisV2Storage.sol#L196-L198](https://github.com/ArrakisFinance/v2-core/blob/376bfcec803f0644fdc601db3a5772d2179c13a0/contracts/abstract/ArrakisV2Storage.sol#L196-L198), the function `setRestrictedMint()` can be frontrun to mint tokens before restricted mint is set since `restrictedMint = address(0)` by default.

### Recommendation

It is recommended to add restricted mint as a vault deployment parameter.

### Status
Acknowledged

## Gas optimization in `renewTerm()`

### Description

In the function `renewTerm()` at the line [PALMTerms.sol#L149](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/PALMTerms.sol#L149), it is possible to use memory variable `manager_` to save gas.

### Recommendation

It is recommended to change to `manager_.renewTerm(address(vault_));`

### Status
Fixed at https://github.com/ArrakisFinance/v2-palm/blob/90e5c17e6c01dcf7158ee4aab82b133bff198726

## User can receive less than expected when decreasing liquidity

### Description

In the function `decreaseLiquidity()` at the lines [PALMTerms.sol#L190-L194](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/PALMTerms.sol#L190-L194).
```solidity
require(
    amount0 >= decreaseBalance_.amount0Min &&
        amount1 >= decreaseBalance_.amount1Min,
    "PALMTerms: received below minimum"
);
```

When calling `decreaseLiquidity()` the user expects to receive `decreaseBalance_.amount0Min`, but actually they receive `amount0 - emolumentAmt0` which might be smaller than `decreaseBalance_.amount0Min` since there is only a check for `amount0`.

### Recommendation

It is recommended to check if `amount0 - emolumentAmt0 >= decreaseBalance_.amount0Min`.

### Status
Fixed at https://github.com/ArrakisFinance/v2-palm/blob/90e5c17e6c01dcf7158ee4aab82b133bff198726

## `termTreasury` can be set to `address(0)`

### Description

At the lines [PALMTermsStorage.sol#L89](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMTermsStorage.sol#L89), [PALMTermsStorage.sol#L113](https://github.com/ArrakisFinance/v2-palm/blob/06f8439430e3b0af9cbbf887926ff93844c28a7d/contracts/abstracts/PALMTermsStorage.sol#L113).

The variable `termTreasury` can be set to `address(0)`.

### Recommendation

It is recommended to ensure `termTreasury` can never be `address(0)`.

### Status
DOUBLE
