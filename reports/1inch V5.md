# CRITICAL

Not found

# HIGH

Not found

# MEDIUM

Not found

# INFORMATIONAL

## Unclear selectors

### Description

At the line https://github.com/1inch/1inch-contract/blob/47f1bc6b5d715efc2d9a8af2d20987ed71722d02/contracts/routers/UnoswapV3Router.sol#L26, the `_SELECTORS` constant stores selectors for `token0()`, `token1()`, `fee()`, `transfer()`, `transferFrom()` functions, but it is not immediately clear which selectors are stored in the variable.

### Recommendation

It is recommended to add a comment to improve readability.

### Status
Fixed at https://github.com/1inch/1inch-contract/commit/21948619f9c2c351e31d476b1cfef0f7bf47091d

## Indirect inheritance of `receive()`

### Description

At the line https://github.com/1inch/limit-order-protocol/blob/d8437885744543e3f057e84e1b0a05c4c211c553/contracts/OrderRFQMixin.sol#L231, the contract `OrderRFQMixin` receives ETH from the `WETH` contract, but it doesn't have a `receive()` function. `AggregationRouterV5` inherits `EthReceiver` from router contracts, so it is not an issue for it, but it might be for other contracts in the future that inherit from `OrderRFQMixin`.

### Recommendation

It is recommended to inherit `EthReceiver` in the `OrderRFQMixin` contract.

### Status
Fixed at https://github.com/1inch/solidity-utils/pull/31, https://github.com/1inch/solidity-utils/commit/68ddec774e908a868f15991c33c978b8e65bfe17, https://github.com/1inch/limit-order-protocol/pull/158, https://github.com/1inch/1inch-contract/pull/161

## Variable shadowing

### Description

At the line https://github.com/1inch/limit-order-protocol/blob/d8437885744543e3f057e84e1b0a05c4c211c553/contracts/helpers/PredicateHelper.sol#L102, the variable `nonce` shadows the state variable `nonce` inherited from the `NonceManager` contract.

### Recommendation

It is recommended to rename the variable to avoid shadowing.

### Status
Fixed at https://github.com/1inch/limit-order-protocol/commit/a0adebcef130b72ee9f7f50125e77077465444d3

## Missing NatSpec comments

### Description

At the lines:
- https://github.com/1inch/limit-order-protocol/blob/d8437885744543e3f057e84e1b0a05c4c211c553/contracts/interfaces/IOrderMixin.sol#L73
- https://github.com/1inch/limit-order-protocol/blob/d8437885744543e3f057e84e1b0a05c4c211c553/contracts/interfaces/IOrderMixin.sol#L98
- https://github.com/1inch/limit-order-protocol/blob/d8437885744543e3f057e84e1b0a05c4c211c553/contracts/interfaces/IOrderMixin.sol#L121

There are no NatSpec comments for parameter `interaction`.

At the line https://github.com/1inch/1inch-contract/blob/47f1bc6b5d715efc2d9a8af2d20987ed71722d02/contracts/routers/GenericRouter.sol#L41, there is no NatSpec comment for parameter `permit`.

### Recommendation

It is recommended to add NatSpec comments where needed.

### Status
Fixed at https://github.com/1inch/1inch-contract/pull/158, https://github.com/1inch/limit-order-protocol/pull/156

## `TODO` comment in the source code

### Description

At the line https://github.com/1inch/limit-order-protocol/blob/d8437885744543e3f057e84e1b0a05c4c211c553/contracts/interfaces/NotificationReceiver.sol#L6, there is a `TODO` comment.

### Recommendation

It is recommended to remove `TODO` comments before deployment.

### Status
Fixed at https://github.com/1inch/limit-order-protocol/commit/c1aaf55f61c37c0b5e497327fd19476d6b7480cf

## Typo in the function name

### Description

At the line https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/UniERC20.sol#L62, there is a typo in `"SYBMOL()"`.

### Recommendation

It is recommended to change to `"SYMBOL()"`.

### Status
Fixed at https://github.com/1inch/solidity-utils/commit/2d8931d48372feb02c99cc9cef2bbe32f1c35aec

## Gas optimization by using constant selectors

### Description

At the lines:
- https://github.com/1inch/1inch-contract/blob/47f1bc6b5d715efc2d9a8af2d20987ed71722d02/contracts/routers/ClipperRouter.sol#L102-L103
- https://github.com/1inch/1inch-contract/blob/47f1bc6b5d715efc2d9a8af2d20987ed71722d02/contracts/routers/ClipperRouter.sol#L132
- https://github.com/1inch/1inch-contract/blob/47f1bc6b5d715efc2d9a8af2d20987ed71722d02/contracts/routers/ClipperRouter.sol#L157
- https://github.com/1inch/1inch-contract/blob/47f1bc6b5d715efc2d9a8af2d20987ed71722d02/contracts/routers/ClipperRouter.sol#L182
- https://github.com/1inch/1inch-contract/blob/47f1bc6b5d715efc2d9a8af2d20987ed71722d02/contracts/routers/ClipperRouter.sol#L208-L209
- https://github.com/1inch/1inch-contract/blob/47f1bc6b5d715efc2d9a8af2d20987ed71722d02/contracts/routers/ClipperRouter.sol#L231
- https://github.com/1inch/limit-order-protocol/blob/d8437885744543e3f057e84e1b0a05c4c211c553/contracts/OrderMixin.sol#L310
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/SafeERC20.sol#L21
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/SafeERC20.sol#L41
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/SafeERC20.sol#L48-L50
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L110
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L129
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L152
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L174

It is possible to save gas by using constant selectors.

### Recommendation

It is recommended to use constant selectors to save gas.

### Status
Acknowledged

### Client's comments
> We decided to use Contract.function.selector notation instead of 4 byte constant to increase readability and rely on compile-time checks

## Gas optimization by removing `len` variable

### Description

At the lines:
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L114
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L133
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L156
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L178

The `len` variable is unnecessary. It can be precomputed and written directly or computed inside the `staticcall()`.

### Recommendation

It is recommended to remove the `len` variable.

### Status
Fixed at https://github.com/1inch/solidity-utils/commit/1be62a2db18fd5260d8e71a6eb53c4487f7a06cc

## Gas optimization by using assembly for external calls

### Description

At the lines:
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/SafeERC20.sol#L58
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/SafeERC20.sol#L64

In the contract `SafeERC20`, the external calls are written in assembly to save gas, except when getting allowance `token.allowance(address(this), spender)`. It is possible to save gas by doing these calls in assembly as well.

At the lines:
- https://github.com/1inch/1inch-contract/blob/47f1bc6b5d715efc2d9a8af2d20987ed71722d02/contracts/routers/UnoswapV3Router.sol#L101
- https://github.com/1inch/1inch-contract/blob/47f1bc6b5d715efc2d9a8af2d20987ed71722d02/contracts/routers/UnoswapV3Router.sol#L117

It is possible to save gas by making external calls to `_WETH` in assembly.

### Recommendation

It is recommended to make external calls in assembly where possible.

### Status
Acknowledged

### Client's comments
> We don't really use safeIncreaseAllowance and safeDecreaseAllowance, so won't fix

## Unnecessary imports

### Description

At the lines:
- https://github.com/1inch/limit-order-protocol/blob/d8437885744543e3f057e84e1b0a05c4c211c553/contracts/OrderMixin.sol#L17
- https://github.com/1inch/limit-order-protocol/blob/d8437885744543e3f057e84e1b0a05c4c211c553/contracts/helpers/AmountCalculator.sol#L6

The imported library `Callib` is not used.

At the line https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/UniERC20.sol#L7, the imported library `RevertReasonForwarder` is not used.

### Recommendation

It is recommended to remove these imports.

### Status
Fixed at https://github.com/1inch/solidity-utils/commit/78586c6c7e4c4255c09f83c272e9ed22134e3ba6, https://github.com/1inch/limit-order-protocol/commit/2034c99db9da20834359d02017fc863c0c89cb53

## Unused error

### Description

At the line https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/UniERC20.sol#L18, there is an unused error `ERC20OperationFailed`.

### Recommendation

It is recommended to remove this error.

### Status
Fixed at https://github.com/1inch/solidity-utils/commit/105da76070e95f91cbe508a877cd927e1553c518

## Missing address zero check

### Description

At the lines:
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L80
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L87
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L94
- https://github.com/1inch/solidity-utils/blob/eec6b523860af5215a8dd196fe3aff3a4d252fc9/contracts/libraries/ECDSA.sol#L101

In the functions `recoverOrIsValidSignature`, if the address `signer == address(0)` and `recover()` returns `address(0)`, then the functions return `true`.

### Recommendation

Consider returning `false` in this case.

### Status
Fixed at https://github.com/1inch/solidity-utils/pull/33

## Unnecessary return statement

### Description

At the line https://github.com/1inch/1inch-contract/blob/47f1bc6b5d715efc2d9a8af2d20987ed71722d02/contracts/routers/ClipperRouter.sol#L256, the function `clipperSwapTo()` returns `outputAmount`, but the value `outputAmount` is taken as a function parameter and it is not changed inside the function making the return statement unnecessary.

### Recommendation

Consider removing the return statement to save gas.

### Status
Acknowledged

### Client's comments
> Won't fix