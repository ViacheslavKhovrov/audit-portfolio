# CRITICAL

## Ability to change the implementation of the wallet during the upgrade from v2 to v3

### Description

In [AvoCore.sol](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoCore/AvoCore.sol) some restrictions are enforced through the usage of `_transientAllowHash` and `_transientId`. There is manipulation of this variables during the upgrade process from V2 version.
Here is the plan:

All actions performed by the owner (they should be done in the same block):

1. Calling the [`cast`](https://etherscan.io/address/0x3718f4bf9140f333bca79cb279f09f0bb8e6ddee#code#F1#L161) function to modify the slot 53 where `_transientAllowHash` is stored with value equal to [`bytes31(keccak256(abi.encode(data_, block.timestamp, EXECUTE_OPERATION_SELECTOR)))`](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoCore/AvoCore.sol#L640) and `_transientId` equal to `1` (to enable `delegatecall` actions).

2. Calling [`upgradeTo(V3Wallet)`](https://etherscan.io/address/0x3718f4bf9140f333bca79cb279f09f0bb8e6ddee#code#F9#L27)

Now we have an access to `executeOperation` function.

3. Calling [`executeOperation`](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoCore/AvoCore.sol#L629) and change slot0 that stores the implementation.

We avoid storage slot0 snapshot checking, because [`isFlashloanCallback_ == true`](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoCore/AvoCore.sol#L651).

In the final step, we can set any implementation. For example, without any fees or set the smart contract as the owner.
The same process could be done if there would be possibility to change versions to old ones.

### Recommendation

We recommend resetting `_transientAllowHash` and `_transientId` during the upgrade process and taking snapshots even if it is `flashloan callback`.

### Status

DOUBLE

# HIGH

## One signer can takeover the multisig

### Description

After contract deployment, the multisig has one signer who is the owner. The owner can add signers to the multisig using [`addSigners()`](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultisig/AvoMultisig.sol#L355), but it doesn't update `requiredSigners`. If the owner does not call [`setRequiredSigners()`](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultisig/AvoMultisig.sol#L503) within the same `cast` as `addSigners()`, this can lead to one added signer to takeover the multisig, since `requiredSigners == 1`. A malicious signer can frontrun any transaction and transfer the funds from the multisig.

The same applies to [`removeSigners()`](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultisig/AvoMultisig.sol#L427). For example, to remove all signers except the owner, `requiredSigners` has to be set to `1` separately from `removeSigners()` call.

### Recommendation

We recommend to add an option to update `requiredSigners` value in the `addSigners()` and `removeSigners()` functions.

### Status

Fixed at https://github.com/instadapp/avocado-contracts/commit/948d196973b683b6495c33290795b96041721a95, https://github.com/instadapp/avocado-contracts/commit/002eefd36cc052a6876dee5e5761aac154bad866

# MEDIUM

## Possible function selector collision

### Description

At the lines [AvoMultiSafe.sol#L65](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultiSafe.sol#L65), [AvoMultiSafe.sol#L72](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultiSafe.sol#L72), [AvoMultiSafe.sol#L83](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultiSafe.sol#L83), [AvoSafe.sol#L53](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoSafe.sol#L53), the proxy contracts check if the calldata matches the selector and return values if it does. However, if the implementation contract has functions with the same function selector, then it will be impossible to call those functions on the implementation.

### Recommendation

We recommend to move these view functions from proxy contracts to implementation or use a transparent proxy pattern.

### Status

Acknowledged

### Client's comments

> this is acceptable to us, but we added comments and tests to make sure we will not accidentally have such methods on the logic contracts: https://github.com/Instadapp/avocado-contracts/pull/162 (https://github.com/instadapp/avocado-contracts/commit/e9d62e1fed8370d6386612e42180fb691abe9ca4)

## A user can make `AvoGasEstimationsHelper` to estimate gas incorrectly

### Description

The contract `AvoGasEstimationsHelper` uses `tx.origin == 0x000000000000000000000000000000000000dEaD` for backend gas estimations. A user can abuse this by checking the `tx.origin` in the `action.target`, i.e. if `tx.origin == 0x000000000000000000000000000000000000dEaD` then it does nothing, otherwise it executes normally. `AvoGasEstimationsHelper` will estimate gas incorrectly, and if this estimation is used for gas payment in the backend, a user can pay less fees.

### Recommendation

We recommend to use the `eth_call` RPC method to evaluate gas usage.

### Status

Acknowledged

### Client's comments

> interesting point. But if user would do this, then the broadcaster would not send enough gas and the tx would fail when actually executed? Also, we don't use the estimated gas fees as payment, but rather the actual tx cost after execution.
> Can't use the eth_call RPC because we want (and need) to estimate before signature. By action.target you simply mean the target contract for the action right?

## Broadcaster can send the cast transaction with a lot more gas than the user instructed

### Description

At the lines [AvoWallet.sol#L372](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoWallet/AvoWallet.sol#L372), [AvoMultisig.sol#L580](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultisig/AvoMultisig.sol#L580), `forwardParams_.gas` is the user instructed minimum amount of gas that the relayer (AvoForwarder) must send for the execution. However, there is no maximum amount of gas that the relayer can send. For example, the user expects `cast` to consume `100_000` gas and sets `forwardParams_.gas = 100_000`. But after the user signed the message and before the broadcaster sends the transaction, due to external factors the cast consumes `1_000_000` gas. The broadcaster sends the transaction with `1_000_000` gas. Then the user has to pay for gas 10 times more than they expected.

It would be better to let the user set the maximum gas they are willing to pay and enforce it on-chain to make sure the broadcasters cannot abuse their role to force users to pay more gas fees.

### Recommendation

We recommend to add a parameter to `CastForwardParams` for maximum gas the relayer is allowed to send.

### Status

Acknowledged

### Client's comments

> We could add the maxGas param in addition to current "gas" (minGas) param.
> There will always be some implicit trust from the user though to work with the unified gas tank that is one of the main features of Avocado.
> The user has to trust the architecture behind it to charge the correct amount in the end. The problem is, user gives instructions for gas amounts on his contract, but part of gas that is also charged is for execution on Forwarder - any check there could be removed (upgradeable)...

# INFORMATIONAL

### Client's comments

> After deployment, AvoFactory always calls initialize() which would fail if this staticcall failed for some reason. That would revert the deployment anyway? Trying to keep deployment cost low with proxy.

## Implicit conversion to `address`

### Description

At the line [AvoCore.sol#L79](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoCore/AvoCore.sol#L79), `sload(_avoImplementation.slot)` takes the implementation address from slot 0. However, the implementation address is packed with other variables in that slot. It is better to explicitly convert slot value to `address` using a bit mask.

### Recommendation

We recommend to change to `and(sload(0), 0xffffffffffffffffffffffffffffffffffffffff)`.

### Status

Fixed at https://github.com/Instadapp/avocado-contracts/commit/1b47b9b4f2249101a8a361e40ba7f55dea663d93

## No revert message

### Description

At the line [AvoCore.sol#L496](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoCore/AvoCore.sol#L496), there is no revert message. Currently, the function `_toHexDigit()` is not used with values `> 15` so it shouldn't revert. However, if it reverts in a future update, it will be useful know why it reverted.

### Recommendation

We recommend to add a custom error for this function to be more consistent with the entire codebase.

### Status

Fixed at https://github.com/Instadapp/avocado-contracts/commit/c753165a9df984c840c46e842402737b21f07495

## Unclear call to `IAvoWalletV2`

### Description

At the line [AvoForwarder.sol#L487](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoForwarder.sol#L487), the function `_callTargets()` is being called if the cast execution failed. However, from the code it is unclear why and which function is being called.

### Recommendation

We recommend to add comments documenting the reason for the call and the function selector.

### Status

Acknowledged

### Client's comments

> We added this code to make sure the status flag on AvoWallets resets for sure even if the action execution via cast reverts.

## Typos

### Description

There are several typos, at the lines:

- [AvoCoreVariables.sol#L54](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoCore/AvoCoreVariables.sol#L54), it should be `address`,
- [AvoWallet.sol#L115](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoWallet/AvoWallet.sol#L115), it should be `signal`,
- [AvoSafe.sol#L31](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoSafe.sol#L31), there is no `hash`.

### Recommendation

We recommend to fix these typos.

### Status

Fixed at https://github.com/Instadapp/avocado-contracts/commit/7b41e7c7af6931bbb59e6b3609fe32d7a2204456

## Magic numbers can be replaced with constant variables

### Description

There are magic numbers which can be replaced with constant variables at the lines:

- [AvoWalletVariables.sol#L39-L40](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoWallet/AvoWalletVariables.sol#L39-L40)
- [AvoMultisigVariables.sol#L50-L51](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultisig/AvoMultisigVariables.sol#L50-L51)
- [AvoMultisig.sol#L260](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultisig/AvoMultisig.sol#L260)
- [AvoMultisig.sol#L270](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultisig/AvoMultisig.sol#L270)

### Recommendation

We recommend to replace magic numbers with constant variables.

### Status

Fixed at https://github.com/Instadapp/avocado-contracts/commit/3a0251416f9e299411aa05c1e3bbeb336dac112a

## Ownable contracts can renounce ownership

### Description

The contracts `AvoAdmin`, `AvoDepositManager`, `AvoForwarder`, `AvoVersionsRegistry` have the function `renounceOwnership()` which can be called to renounce ownership. If it is called by mistake, the contracts will be left without an owner. If the owner is `address(0)`, important functions will not be available.

### Recommendation

We recommend to override the function `renounceOwnership()` and revert if called.

### Status

Fixed at https://github.com/Instadapp/avocado-contracts/commit/b9cd50269bc5c4e8ce8d1ecea510896dc102074a

## No `address(0)` check in `AvoFactory`

### Description

In `AvoFactory`, the modifier [`onlyEOA()`](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoFactory.sol#L136) doesn't check if the address `owner_ == address(0)`. This makes it possible to deploy an `AvoSafe` or `AvoMultiSafe` with `address(0)` as the owner.

### Recommendation

We recommend to check if the `owner` is not `address(0)`.

### Status

Fixed at https://github.com/Instadapp/avocado-contracts/commit/b9cd50269bc5c4e8ce8d1ecea510896dc102074a

## Can upgrade the implementation to the same implementation

### Description

In the functions [`AvoWallet.upgradeTo()`](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoWallet/AvoWallet.sol#L297), [`AvoMultisig.upgradeTo()`](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultisig/AvoMultisig.sol#L321), there is no check if `avoImplementation_` is equal to the current implementation `_avoImplementation`. This makes it possible to upgrade the implementation to the same implementation and delegate call.

### Recommendation

We recommend to check if `avoImplementation_ == _avoImplementation`.

### Status

Fixed at https://github.com/Instadapp/avocado-contracts/commit/13b4209093bbcf38e3186d58c6a4879ea1612853

## Gas optimizations

### Description

1. It is possible to save gas by removing `== true` and replacing `== false` with `!` in `if` conditions:

   - [AvoAuthoritiesList.sol#L179](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoAuthoritiesList.sol#L179)
   - [AvoAuthoritiesList.sol#L190](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoAuthoritiesList.sol#L190)
   - [AvoDepositManager.sol#L170](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoDepositManager.sol#L170)
   - [AvoFactory.sol#L176](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoFactory.sol#L176)
   - [AvoForwarder.sol#L390](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoForwarder.sol#L390)
   - [AvoForwarder.sol#L484](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoForwarder.sol#L484)
   - [AvoForwarder.sol#L547](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoForwarder.sol#L547)
   - [AvoForwarder.sol#L618](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoForwarder.sol#L618)
   - [AvoMultisig.sol#L625](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultisig/AvoMultisig.sol#L625)
   - [AvoMultisig.sol#L734](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoMultisig/AvoMultisig.sol#L734)
   - [AvoWallet.sol#L404](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoWallet/AvoWallet.sol#L404)
   - [AvoWallet.sol#L476](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoWallet/AvoWallet.sol#L476)
   - [AvoSignersList.sol#L146](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoSignersList.sol#L146)
   - [AvoSignersList.sol#L156](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoSignersList.sol#L156)
   - [AvoSignersList.sol#L206](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoSignersList.sol#L206)
   - [AvoSignersList.sol#L224](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoSignersList.sol#L224)
   - [AvoSignersList.sol#L237](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoSignersList.sol#L237)
   - [AvoSignersList.sol#L274](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoSignersList.sol#L274)
2. Replace `i++` with `++i`:

   - [AvoForwarder.sol#L682](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoForwarder.sol#L682)
   - [AvoForwarder.sol#L711](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoForwarder.sol#L711)
   - [AvoForwarder.sol#L762](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoForwarder.sol#L762)

### Recommendation

We recommend to implement these gas optimizations.

### Status

Fixed at https://github.com/Instadapp/avocado-contracts/commit/5f49805a5449577db48c8bba14085ca0c1c887c7

## Missing `isAvoSafe` checks

### Description

1. At the line [AvoSignersList.sol#L93](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoSignersList.sol#L93), [AvoSignersList.sol#L112](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoSignersList.sol#L112), there is a check if `avoMultiSafe_` is a contract, but it would be better to use the `AvoFactory.isAvoSafe()` check.
2. At the line [AvoAuthoritiesList.sol#L91](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoAuthoritiesList.sol#L91), there is no check if `avoSafe_` is deployed by `AvoFactory`.
3. At the lines [AvoAuthoritiesList.sol#L89](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoAuthoritiesList.sol#L89), [AvoAuthoritiesList.sol#L99](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoAuthoritiesList.sol#L99), [AvoAuthoritiesList.sol#L119](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoAuthoritiesList.sol#L119), [AvoSignersList.sol#L85](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoSignersList.sol#L85). If the `AvoSafe` or `AvoMultiSafe` is selfdestructed, it will remove signers and authorities without calling `sync` functions in `AvoAuthoritiesList` and `AvoSignersList`. It will be impossible to call `sync` functions because `AvoFactory` checks if there is code at the address. These functions will return wrong old values.

### Recommendation

We recommend to add `isAvoSafe` checks.

### Status

Fixed at https://github.com/Instadapp/avocado-contracts/commit/76aeb930990d903b9db1e9facf0fdb05306162be

## Broadcaster can send `msg.value` more or less than required in actions

### Description

The broadcaster sets `msg.value` at the line [AvoForwarder.sol#L541](https://github.com/instadapp/avocado-contracts/blob/0eec33a45784e5f4b3b45ac2c9dfefc5225bda0a/contracts/AvoForwarder.sol#L541), to send to `cast`. It is unclear how this value is set by the broadcaster off-chain. If the user pays for `msg.value` by sending Ether to the broadcaster, the broadcaster can accidentally send more than he received and the leftover Ether will stay in the wallet. The broadcaster can set `msg.value = 0`, then the Ether for actions will be used from the user's `AvoWallet` balance. Then the user will pay `msg.value` twice.

### Recommendation

We recommend to add a `CastForwardParams` parameter value set by the user that has to equal `msg.value` set by the broadcaster. The user should choose whether to use `AvoWallet` Ether balance for target actions.

### Status

Fixed at https://github.com/Instadapp/avocado-contracts/commit/73e2f7b46d7583285a161cd66d22add679ecc9c2, https://github.com/Instadapp/avocado-contracts/commit/3f997a9eec14cd19c380d444cc0887996c95b02d, https://github.com/Instadapp/avocado-contracts/commit/3d5e911ba58db003b4ba222b877995483a546755, https://github.com/Instadapp/avocado-contracts/commit/370b44d7a0f2f96c285f6fc7fd48664a8a04442a


### Client's comments

> For this, allowing deposits when paused is intended

## Race condition for 3 required signers out of 6 signers

### Description

At the line [AvoMultisigAdmin.sol#L18](https://github.com/Instadapp/avocado-contracts/blob/eb347612298123e72db594b16b40a81dbe4a3c72/contracts/admin/AvoMultisigAdmin.sol#L18), `AvoMultisigAdmin` has `REQUIRED_CONFIRMATIONS = 3` and 6 signers. This creates a race condition when the signers cannot come to a consensus and the signers are split. For example, 2 signers confirm, 2 signers revoke, then the last 2 signers race to confirm or revoke the transaction.

### Recommendation

We recommend to set `REQUIRED_CONFIRMATIONS = 4`.

### Status

Acknowledged

## Signer can confirm and revoke the same transaction

### Description

At [AvoMultisigAdmin.sol#L119](https://github.com/Instadapp/avocado-contracts/blob/eb347612298123e72db594b16b40a81dbe4a3c72/contracts/admin/AvoMultisigAdmin.sol#L119), the signer can confirm a transaction then revoke the same transaction. For example, if 2 signers confirm and revoke the transaction, the third signer will confirm or revoke the transaction either way.

### Recommendation

We recommend to disallow confirming and revoking the same transaction.

### Status

Acknowledged
