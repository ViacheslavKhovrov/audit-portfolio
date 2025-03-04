# CRITICAL

Not found

# HIGH

Not found

# MEDIUM

## Whitelisting the same pipeline can disable writing to the oracle

### Description

At the lines [contracts/peripherals/PipelineManagement.sol#L139-L144](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/peripherals/PipelineManagement.sol#L139-L144):
```solidity
  function _whitelistPipeline(uint32 _chainId, bytes32 _poolSalt) internal {
    (uint24 _lastPoolNonceObserved, , , ) = IDataFeed(address(this)).lastPoolStateObserved(_poolSalt);
    whitelistedNonces[_chainId][_poolSalt] = _lastPoolNonceObserved + 1;
    _whitelistedPools.add(_poolSalt);
    emit PipelineWhitelisted(_chainId, _poolSalt, _lastPoolNonceObserved + 1);
  }
```

Scenario:
1. A pipeline is whitelisted, `whitelistedNonces[_chainId][_poolSalt] = 1`.
2. A keeper/user fetches and sends observations for the pool with `_poolNonce = 1`. Then `poolNonce = 1` in `OracleSidechain`.
3. The keeper/user fetches observations with `_poolNonce = 2`.
4. The same pipeline is whitelisted again, `whitelistedNonces[_chainId][_poolSalt] = 3`.
5. The keeper/user cannot send observations with `_poolNonce = 2` since `_whitelistedNonce > _poolNonce`.

The observations with `_poolNonce >= 3` can be sent, but they will not be written since `poolNonce = 1` in `OracleSidechain`.

### Recommendation

It is recommended to check if the same pipeline has already been whitelisted.

### Status

Fixed at https://github.com/defi-wonderland/sidechain-oracles/commit/01d002f32dfacafb870bb8ab98f20e73ce80d946


## Bridged nonces are not tracked properly

### Description

At the lines [contracts/StrategyJob.sol#L76-L88](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/StrategyJob.sol#L76-L88):
```solidity
  function _workable(
    uint32 _chainId,
    bytes32 _poolSalt,
    uint24 _poolNonce
  ) internal view returns (bool _isWorkable) {
    uint24 _lastPoolNonceBridged = lastPoolNonceBridged[_chainId][_poolSalt];
    if (_lastPoolNonceBridged == 0) {
      (uint24 _lastPoolNonceObserved, , , ) = dataFeed.lastPoolStateObserved(_poolSalt);
      return _poolNonce == _lastPoolNonceObserved;
    } else {
      return _poolNonce == ++_lastPoolNonceBridged;
    }
  }
```

Scenario 1:
1. A keeper fetches and sends observations for a new pool with `_poolNonce = 1`. Then `lastPoolNonceBridged[_chainId][_poolSalt] = 1`.
2. A user fetches and sends observations for the same pool with `_poolNonce = 2`.
3. The keeper fetches and tries to send observations with `_poolNonce = 3`, but it fails because `_poolNonce != ++_lastPoolNonceBridged`.

The keeper has to send observations with `_poolNonce = 2` that will revert in the sidechain to synchronize nonces.

Scenario 2:
1. A user fetches observations for a new pool three times such that `dataFeed.lastPoolStateObserved(_poolSalt) = 3`.
2. The user sends observations with `_poolNonce = 1`. Then `poolNonce = 1` in `OracleSidechain`.
3. The keeper sends observations with `_poolNonce = 3` with `lastPoolNonceBridged[_chainId][_poolSalt] = 3`, but it reverts in the sidechain because `_poolNonce != ++poolNonce`.

The user has to send observations with `_poolNonce = 2` to synchronize nonces.

### Recommendation

It is recommended to save `lastPoolNonceBridged` in `DataFeed` to keep track of all bridged nonces.

### Status

Acknowledged

### Client's comments
 > The reward layer of the setup (StrategyJob) should be completely independent from the Strategy behaviour. In Scenario 1.2, the keeper will also be able to send observations _poolNonce = 2, despite they will revert in the sidechain (because they were already bridged).
With Scenario 2, where 3 fetch tx are made ‘at the same time’, if a user chooses to bridge n3 first, then n1 and n2 (and again n3) are also going to be bridged (keepers will work it), but revert on the sidechain.


# INFORMATIONAL

## `TODO` comments

### Description

At the lines:
- [contracts/DataFeedStrategy.sol#L168](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/DataFeedStrategy.sol#L168)
- [contracts/StrategyJob.sol#L40](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/StrategyJob.sol#L40)
- [contracts/bridges/ConnextSenderAdapter.sol#L25](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/bridges/ConnextSenderAdapter.sol#L25)
- [interfaces/IOracleSidechain.sol#L16](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/interfaces/IOracleSidechain.sol#L16)

`TODO` comments are irrelevant and should be removed before deployment.

### Recommendation

It is recommended to remove `TODO` comments.

### Status

Fixed at https://github.com/defi-wonderland/sidechain-oracles/commit/044a239ea1ea9e65590ca6ce878a245b305fa462


## Period duration can be set to `0`

### Description

At the line [contracts/DataFeedStrategy.sol#L193](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/DataFeedStrategy.sol#L193), there is no check if `_periodDuration` is bigger than `0`.

### Recommendation

It is recommended to check if `_periodDuration > 0`.

### Status

Fixed at https://github.com/defi-wonderland/sidechain-oracles/commit/713879b63ac51421376de396da05ad9bc03f2685


## Unused constant variable `ORACLE_INIT_CODE_HASH`

### Description

At the line [contracts/DataReceiver.sol#L21](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/DataReceiver.sol#L21), the constant variable `ORACLE_INIT_CODE_HASH` is unused in the contract.

### Recommendation

It is recommended to remove the unused variable.

### Status
DOUBLE

## Helper functions for pool salt

### Description

1. Currently, in the contracts a pool salt is used to identify a UniswapV3 pool. For users to calculate pool salt they have to get `keccak256(abi.encode(token0, token1, fee)` externally. There is [`getPoolSalt()`](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/OracleFactory.sol#L72) in `OracleFactory.sol` already, but it would be useful to have it also on the mainnet side.

2. To use [`initializePoolInfo()`](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/OracleSidechain.sol#L72) in `OracleSidechain.sol` users have to know `token0, token1, fee` to match the `poolSalt`. It would be useful to have a helper function that given the pool salt would return these values from the UniswapV3 pool.

### Recommendation

It is recommended to add helper functions.

### Status

Acknowledged

### Client's comments
> keepers should listen to the events and push the emitted data, not caring about which pool it corresponds to. Users that choose to initializePoolInfo are users of the Oracle, that found it by querying the Factory with token0, token1, fee, so if required, they would know what parameters to input to initialize the pool info.

## `bridgeObservations()` is `payable` without Ether transfer

### Description

At the line [contracts/bridges/ConnextSenderAdapter.sol#L26](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/bridges/ConnextSenderAdapter.sol#L26), the function `bridgeObservations()` is `payable` without receiving Ether and transferring Ether.

### Recommendation

It is recommended to remove the `payable` modifier.

### Status

DOUBLE


## No way to remove a whitelisted pipeline

### Description

At the lines [contracts/peripherals/PipelineManagement.sol#L139-L144](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/peripherals/PipelineManagement.sol#L139-L144). In the contract `PipelineManagement.sol` there is no way to remove a whitelisted pipeline if there is a need to remove a pipeline.

### Recommendation

It is recommended to add a way to remove a whitelisted pipeline.

### Status
DOUBLE

## Missing NatSpec comments

### Description

At the lines:
- [interfaces/bridges/IBridgeReceiverAdapter.sol#L10](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/interfaces/bridges/IBridgeReceiverAdapter.sol#L10)
- [interfaces/bridges/IBridgeSenderAdapter.sol#L9](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/interfaces/bridges/IBridgeSenderAdapter.sol#L9)
- [interfaces/bridges/IConnextReceiverAdapter.sol#L10-L14](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/interfaces/bridges/IConnextReceiverAdapter.sol#L10-L14)
- [interfaces/bridges/IConnextSenderAdapter.sol#L11-L13](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/interfaces/bridges/IConnextSenderAdapter.sol#L11-L13)
- [interfaces/peripherals/IPipelineManagement.sol#L10-L87](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/interfaces/peripherals/IPipelineManagement.sol#L10-L87)

There are missing NatSpec comments. 

### Recommendation

It is recommended to add NatSpec comments.

### Status

Fixed at https://github.com/defi-wonderland/sidechain-oracles/commit/6723d152ab15ff19e7bc09534a5f0fcb90311305


## Event parameters can be `indexed`

### Description

At the lines [interfaces/peripherals/IPipelineManagement.sol#L20-L26](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/interfaces/peripherals/IPipelineManagement.sol#L20-L26), the parameters `_chainId`, `_bridgeSenderAdapter`, `_destinationDomainId`, `_dataReceiver` can be indexed.

At the line [interfaces/IDataFeed.sol#L51](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/interfaces/IDataFeed.sol#L51), the parameters `_chainId`, `_dataReceiver` can be indexed.

At the lines [interfaces/IDataReceiver.sol#L36-L46](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/interfaces/IDataReceiver.sol#L36-L46), the parameters `_receiverAdapter`, `_adapter` can be indexed.

At the line [interfaces/IOracleSidechain.sol#L80](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/interfaces/IOracleSidechain.sol#L80), the parameters `token0`, `token1` can be indexed.

### Recommendation

It is recommended to change event parameters to `indexed` where appropriate.

### Status

Acknowledged

### Client's comments
> indexed parameters are good to filter and search for specific events, but are hard to read from those events. Keepers need to read the information emitted, so that they can broadcast the data.


## Functions can be `external`

### Description

At the lines:
- [contracts/peripherals/Keep3rJob.sol#L12](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/peripherals/Keep3rJob.sol#L12)
- [contracts/StrategyJob.sol#L57](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/StrategyJob.sol#L57)
- [contracts/StrategyJob.sol#L72](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/StrategyJob.sol#L72)

The functions can be made `external` to optimize gas. 

### Recommendation

It is recommended to change these functions to `external`.

### Status

Fixed at https://github.com/defi-wonderland/sidechain-oracles/commit/a73230ce7711973af3131b60d2c32f2e80c1a480


## `slot0.unlocked` in `OracleSidechain` has opposite meaning

### Description

At the lines [contracts/OracleSidechain.sol#L67](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/OracleSidechain.sol#L67), [contracts/OracleSidechain.sol#L77](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/OracleSidechain.sol#L77), [contracts/OracleSidechain.sol#L85](https://github.com/defi-wonderland/sidechain-oracles/blob/da7cf7d15fca848828f3a2c6e0e8c55e0dd76841/solidity/contracts/OracleSidechain.sol#L85).

The variable `slot0.unlocked` is set to `true` in the constructor and then to `false` after initialization. This means that the oracle is "locked" after initialization, but it should be "unlocked".

### Recommendation

It is recommended to set `slot0.unlocked = false` in the constructor and then to `true` after initialization. Change the check to 
```solidity
if (slot0.unlocked) revert AI();
```

### Status

Acknowledged

### Client's comments
> locked means that the method to be called is unaccesible (because it’s locked), while unlocked means the opposite.
An oracle is locked when it already has the initialized variables. slot0.unlocked was used for this to keep UniV3 slot0, none of the non-reentrant calls from UniV3 are present in the contract.

## `minLastOracleDelta` and `periodDuration` are independent from each other

### Description

At the lines:
- [contracts/DataFeed.sol#L116](https://github.com/defi-wonderland/sidechain-oracles/blob/77afdbdec6a42f91f0f5436118fcb329c3431669/solidity/contracts/DataFeed.sol#L116)
- [contracts/DataFeed.sol#L153](https://github.com/defi-wonderland/sidechain-oracles/blob/77afdbdec6a42f91f0f5436118fcb329c3431669/solidity/contracts/DataFeed.sol#L153)
- [contracts/DataFeedStrategy.sol#L240](https://github.com/defi-wonderland/sidechain-oracles/blob/77afdbdec6a42f91f0f5436118fcb329c3431669/solidity/contracts/DataFeedStrategy.sol#L240)

If `minLastOracleDelta` is set to be bigger than `periodDuration` then every `fetchObservations()` would revert. 
Alternatively, if `periodDuration` is set to be smaller than `minLastOracleDelta` then every `fetchObservations()` would revert.

### Recommendation

It is recommended to check that `minLastOracleDelta` cannot be set to be bigger than `periodDuration` and `periodDuration` cannot be set to be smaller than `minLastOracleDelta`.

### Status

Acknowledged

