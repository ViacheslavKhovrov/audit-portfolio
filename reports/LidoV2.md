# CRITICAL

# HIGH

## The function `balanceOf()` includes claimed requests

### Description

The function [`WithdrawalRequestNFT.balanceOf()`](https://github.com/lidofinance/lido-dao/blob/89ad4cc35407609081afb94df535d71b27bb44b7/contracts/0.8.9/WithdrawalRequestNFT.sol#L114) includes claimed requests. After the request is claimed the request is not removed from `_getRequestsByOwner()`. The function [`WithdrawalRequestNFT.ownerOf()`](https://github.com/lidofinance/lido-dao/blob/89ad4cc35407609081afb94df535d71b27bb44b7/contracts/0.8.9/WithdrawalRequestNFT.sol#L122) reverts if the request is claimed, so the user has balance without being an owner.

### Recommendation

We recommend to include only unclaimed requests in `WithdrawalRequestNFT.balanceOf()`.

### Status

DOUBLE

### Client's comments

> Exit order doesn't presume beginning from 0 and by-1 increments because this code uses a validator index defined on the Ethereum CL side (https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/validator.md#validator-index)
> Therefore we can assume the following sequence of exits:
NO1VAL1 -> NO1VAL3 -> NO2VAL2 -> NO2VAL5 -> NO3VAL4 -> NO3VAL9 -> NO4VAL6 -> NO4VAL8

## A Node Operator can circumvent DAO validator key approval

### Description

This issue follows from a prior issue [https://github.com/lidofinance/lido-dao/issues/141](https://github.com/lidofinance/lido-dao/issues/141) but with different steps:

> Adding validator keys is performed as follows:
>
>  1. A node operator appends new keys to its set; the node operator's staking limit stays the same, making the added keys unusable yet.
>  2. The DAO performs off-chain verification of the submitted keys: it checks that withdrawal credentials, deposit amounts, and signatures are correct and that there are no duplicate keys in the updated key set.
>  3. The DAO votes to increase the node operator's limit to allow the added keys to be used by the pool.

However, a node operator can currently add keys that were not approved by the DAO using a simple trick:

1. A node operator submits `N` correct keys and then enacts an Easy Track motion to increase the staking limit.
2. The DAO validates the keys and votes for the motion with `stakingLimit = N`.
3. After the motion is passed the node operator submits `M <= N` more (bad) keys and removes the first `M` keys.
4. The node operator can enact the motion or front run the transaction enacting the motion.
5. The motion is enacted and sets the `stakingLimit = N` and the newly-added keys are allowed to be used in staking despite not being approved by the DAO.

### Recommendation

A possible solution is to save a different nonce for each node operator and vote for motion with expected nonce, then check if the nonces match when enacting the motion.

### Status

Acknowledged

### Client's comments

> At the launch of the StakingRouter, it will be used only with the Curated staking module (NodeOperatorsRegistry). All node operators registered in the NodeOperatorsRegistry pass strict selection and provide the best service on the market. For professional node operators, such an attack will lead to reputational damage in the short term and financial losses in the long term.
> But even though the opportunity of the described attack is very low, Lido makes the below action to minimize the damage:
> - Constantly monitoring offchain uploaded/deposited keys by their node operators. If the described attack happens, Lido's alerting system will allow it to react fastly and pause deposits via `DepositSecurityModule` until the malicious node operator will be deactivated.
> - the `DepositSecurityModule` has the upper bound for the number of deposits made in one transaction (`maxDepositsPerBlock`) and the cooldown between deposits for the same staking module. It gives enough time to react to pause deposits in response to the malicious actions of node operators.
> The fix for the highlighted issue is planned at the next update of the protocol and definitely will be eliminated before the introduction of a new staking module with a lower trust level for the node operators.

# MEDIUM

## Wrong `burntWithdrawalsShares` calculation

### Description

The simulated share rate is calculated by simulating an `eth_call` with no finalization of withdrawals. The burner contract can have postponed shares to burn from previous oracle reports.

The function [`_burnSharesLimited()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.4.24/Lido.sol#L1359):

```solidity
if (_sharesToBurnLimit > 0) {
    uint256 sharesCommittedToBurnNow = _burner.commitSharesToBurn(_sharesToBurnLimit);

    if (sharesCommittedToBurnNow > 0) {
        _burnShares(address(_burner), sharesCommittedToBurnNow);
    }
}

(uint256 coverShares, uint256 nonCoverShares) = _burner.getSharesRequestedToBurn();
uint256 postponedSharesToBurn = coverShares.add(nonCoverShares);

burntCurrentWithdrawalShares =
    postponedSharesToBurn < _sharesToBurnFromWithdrawalQueue ?
    _sharesToBurnFromWithdrawalQueue - postponedSharesToBurn : 0;
```

In the real call `sharesToBurnLimit` increases compared to the simulated call. This can lead to burning of postponed shares in the real call which was not predicted in the simulated call, but `burntWithdrawalsShares = 0` in both calls.

Consider a case:

```solidity
// Before burn request:
coverSharesBurnRequested = 1000;
nonCoverSharesBurnRequested = 3000;
// Simulated call:
sharesToBurnLimit = 1000;
// After burn commit with sharesToBurnLimit = 1000:
coverSharesBurnRequested = 0;
nonCoverSharesBurnRequested = 3000;
sharesToBurnNow = 1000;

// Real call:
sharesToBurnLimit = 3000;
// After burn request of 1000 shares:
coverSharesBurnRequested = 1000;
nonCoverSharesBurnRequested = 3000 + 1000;
// After burn commit with sharesToBurnLimit = 3000:
coverSharesBurnRequested = 0;
nonCoverSharesBurnRequested = 2000;
sharesToBurnNow = 3000;
postponedSharesToBurn = 2000;
burntWithdrawalsShares = 0;
// but actually 1000 cover and 2000 non cover shares were burnt in a real call
// compared to 1000 cover shares in a simulated call
```

This can lead to reverts when the simulated share rate is checked by the sanity checker contract.

### Recommendation

We recommend accounting for `burntWithdrawalsShares` computed during the simulation call.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/117b0fa33f76e96cf41a3fa6fcd6cde15f2a66de

## Malicious staking module can disable updating exited validator counts for all modules

### Description

The function [`onValidatorsCountsByNodeOperatorReportingFinished()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L429-L433) is called from the `AccountingOracle` when submitting the report extra data which includes exited and stuck validator counts. If one staking module reverts then it becomes impossible to update exited and stuck validator counts.

If the oracle submits empty extra data [`_submitReportExtraDataEmpty()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/oracle/AccountingOracle.sol#L681), then the function `onValidatorsCountsByNodeOperatorReportingFinished()` makes unnecessary external calls to every staking module.

### Recommendation

We recommend to make external calls only to modules which exited and stuck validators counts have been updated.

### Status

DOUBLE

## Possible to set stuck/exited validators count to be bigger than deposited

### Description

The oracle extra data report sets the stuck validators for staking module before setting exited validators count at the lines [AccountingOracle.sol#L854-L860](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/oracle/AccountingOracle.sol#L854-L860). This leads to a situation where it is possible to stuck and exited validators count to be bigger than deposited for a staking module due to a wrong check.

At the line [`NodeOperatorsRegistry._updateStuckValidatorsCount()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L615-L618), `_stuckValidatorsCount` is checked with a value exited count not yet updated.

```solidity
_requireValidRange(
    _stuckValidatorsCount
        <= signingKeysStats.get(DEPOSITED_KEYS_COUNT_OFFSET) 
            - signingKeysStats.get(EXITED_KEYS_COUNT_OFFSET)
);
```

Then during [`NodeOperatorsRegistry._updateExitedValidatorsCount()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L568), `_exitedValidatorsKeysCount` is only checked to be smaller than deposited count without taking into consideration the stuck amount.

```solidity
_requireValidRange(_exitedValidatorsKeysCount <= signingKeysStats.get(DEPOSITED_KEYS_COUNT_OFFSET));
```

### Recommendation

We recommend fixing the check in [`NodeOperatorsRegistry._updateExitedValidatorsCount()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L568) to such that exited and stuck counts in total cannot exceed deposited.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/e4d0bf7d534ae0d6ceb58afdd045be1dfaf09658

## Wrong offsets for stuck and refunded validators

### Description

At the line [NodeOperatorsRegistry.sol#L614](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L614), `STUCK_VALIDATORS_COUNT_OFFSET` should be used.

```solidity
uint64 curStuckValidatorsCount = stuckPenaltyStats.get(REFUNDED_VALIDATORS_COUNT_OFFSET);
```

At the line [NodeOperatorsRegistry.sol#L623](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L623), `REFUNDED_VALIDATORS_COUNT_OFFSET` should be used.

```solidity
uint64 curRefundedValidatorsCount = stuckPenaltyStats.get(STUCK_VALIDATORS_COUNT_OFFSET);
```

### Recommendation

We recommend using correct offsets.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/e470fca0136b7ddf1c621df5de805640c288ffcf

### Client's comments

> The described finding is incorrect. Please recheck the withdrawal finalization flow because discounted batches check happens in the `WithdrawalQueueBase.prefinalize()` method, which returns `_amountOfETH` to pass into `WithdrawalQueueBase._finalize()`. Some ether might stay on the contract only when Oracle reports invalid batches.
> Though, there is an opportunity that oracle would deliver the wrong finalization batches positions but this finding is shaped differently.

## Possible to submit report extra data in the next frame after the report

### Description

When submitting extra data, the function [`_checkCanSubmitExtraData()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.8.9/oracle/AccountingOracle.sol#L695) checks the processing deadline [`_checkProcessingDeadline()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.8.9/oracle/BaseOracle.sol#L289).

```solidity
function _checkCanSubmitExtraData(ExtraDataProcessingState memory procState, uint256 format)
    internal view
{
    _checkMsgSenderIsAllowedToSubmitData();
    _checkProcessingDeadline(); // missing check for extra data ref slot
    
    if (procState.refSlot != LAST_PROCESSING_REF_SLOT_POSITION.getStorageUint256()) {
        revert CannotSubmitExtraDataBeforeMainData();
    }

    if (procState.dataFormat != format) {
        revert UnexpectedExtraDataFormat(procState.dataFormat, format);
    }
}
```

But it is possible to submit report extra data in the next frame after the main report in a certain scenario:

1. The main report data is submitted for the current frame.
2. The extra data is not submitted for the current frame.
3. In the next frame, the consensus contract submits a report which updates the `_storageConsensusReport()` value and emits the event `WarnExtraDataIncompleteProcessing`.
4. The report's extra data for the previous frame can be submitted before the processing of the new report.

### Recommendation

We recommend checking the extra data ref slot with the current ref slot when submitting extra data.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/8488ee9b56e7a812f85a291dac83a7ce5497629f

## No check for return value size in `isValidSignature()`

### Description

At the line [SignatureUtils.sol#L41-L53](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/common/lib/SignatureUtils.sol#L41-L53):

```solidity
bytes4 retval;
/// @solidity memory-safe-assembly
assembly {
    // allocate memory for storing the return value
    let outDataOffset := mload(0x40)
    mstore(0x40, add(outDataOffset, 32))
    // issue a static call and load the result if the call succeeded
    let success := staticcall(gas(), signer, add(data, 32), mload(data), outDataOffset, 32)
    if eq(success, 1) {
        retval := mload(outDataOffset)
    }
}
return retval == ERC1271_IS_VALID_SIGNATURE_SELECTOR;
```

`retval` takes the first 4 bytes of the return value without checking the size of the return value. If the `staticcall` returns more than 4 bytes with the first 4 bytes equalling `ERC1271_IS_VALID_SIGNATURE_SELECTOR`, then the function `isValidSignature()` will return `true`.

### Recommendation

We recommend checking if the return data size is 4 bytes.

### Status

Fixed at https://github.com/lidofinance/commit/2dbad61c907b61ecd85d0f1463aa6c9f4c810c0a

## `NodeOperatorsRegistry` keys nonce doesn't change

### Description

The keys nonce should change when a node operator's ready-to-deposit data size is changed. The function [`_applyNodeOperatorLimits()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L774) can change ready-to-deposit data size:

```solidity
function _applyNodeOperatorLimits(uint256 _nodeOperatorId) internal returns (int64 maxSigningKeysDelta) {
    Packed64x4.Packed memory signingKeysStats = _loadOperatorSigningKeysStats(_nodeOperatorId);
    Packed64x4.Packed memory operatorTargetStats = _loadOperatorTargetValidatorsStats(_nodeOperatorId);

    uint64 exitedSigningKeysCount = signingKeysStats.get(EXITED_KEYS_COUNT_OFFSET);
    uint64 depositedSigningKeysCount = signingKeysStats.get(DEPOSITED_KEYS_COUNT_OFFSET);
    uint64 vettedSigningKeysCount = signingKeysStats.get(VETTED_KEYS_COUNT_OFFSET);

    uint64 oldMaxSigningKeysCount = operatorTargetStats.get(MAX_VALIDATORS_COUNT_OFFSET);
    uint64 newMaxSigningKeysCount = depositedSigningKeysCount;

    if (isOperatorPenaltyCleared(_nodeOperatorId)) {
        if (operatorTargetStats.get(IS_TARGET_LIMIT_ACTIVE_OFFSET) == 0) {
            newMaxSigningKeysCount = vettedSigningKeysCount;
        } else {
            // correct max count according to target if target is enabled
            // targetLimit is limited to UINT64_MAX
            uint256 targetLimit = Math256.min(
                    uint256(exitedSigningKeysCount).add(operatorTargetStats.get(TARGET_VALIDATORS_COUNT_OFFSET)), 
                    UINT64_MAX
                );
            if (targetLimit > depositedSigningKeysCount) {
                newMaxSigningKeysCount = uint64(Math256.min(vettedSigningKeysCount, targetLimit));
            }
        }
    } // else newMaxSigningKeysCount = depositedSigningKeysCount, so depositable keys count = 0

    if (oldMaxSigningKeysCount != newMaxSigningKeysCount) {
        operatorTargetStats.set(MAX_VALIDATORS_COUNT_OFFSET, newMaxSigningKeysCount);
        _saveOperatorTargetValidatorsStats(_nodeOperatorId, operatorTargetStats);
        maxSigningKeysDelta = int64(newMaxSigningKeysCount) - int64(oldMaxSigningKeysCount);
    }
}
```

However, there are cases when the nonce doesn't change:

- [`updateStuckValidatorsCount()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L451), if the operator is penalized its depositable keys size is set to `0`
- [`updateExitedValidatorsCount()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L489), when the exited count is updated the target limit changes which can change the depositable keys size
- [`unsafeUpdateValidatorsCount()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L549), follows from above
- [`updateTargetValidatorsLimits()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L594) changes the target limit
- [`clearNodeOperatorPenalty()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L1225) clears the penalty and changes depositable keys size

### Recommendation

We recommend changing the nonce in these functions.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/26844cf294c680f50c76d3c1e1b0c21912f99c49

# INFORMATIONAL

## `clearNodeOperatorPenalty()` can be restricted to `external`

### Description

The function [`clearNodeOperatorPenalty()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L1225) is not called internally, so it be restricted to `external`.

### Recommendation

We recommend making the function `external`.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/6f37ff5cf37a6226c5209761ffbb578bd597300d

## Unsafe memory handling in `getStakingRewardsDistribution()`

### Description

At the line [StakingRouter.sol#L944](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.8.9/StakingRouter.sol#L944), `stakingModuleIds` is trimmed to length `rewardedStakingModulesCount`. At the line [StakingRouter.sol#L911](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.8.9/StakingRouter.sol#L911), `stakingModuleIds` will be updated regardless if the last staking module has no active validators. Then when the array is trimmed it will leave dirty bytes in memory.

### Recommendation

We recommend moving the line [StakingRouter.sol#L911](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.8.9/StakingRouter.sol#L911) inside the `if` statement.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/48e3a54b35a8af699ac1d068052ada757e35f5d7

## Request owner should not be `payable`

### Description

At the line [WithdrawalQueueERC721.sol#L233](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.8.9/WithdrawalQueueERC721.sol#L233), `request.owner` is set to `payable(_to)`. In the `WithdrawalQueueBase` the `owner` is not `payable`, and recipient is `payable` during claiming. So it is unnecessary to set `owner` as `payable` when transferring.

### Recommendation

We recommend removing `payable`.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/1e17bad7eb4a152821196bd201644c954bdb06e7

## Inconsistent input string for role hash

### Description

At the line [OracleReportSanityChecker.sol#L104](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol#L104), the name of the role `ALL_LIMITS_MANAGER_ROLE` doesn't match the input string for hash `LIMITS_MANAGER_ROLE`. However, for the other roles these match.

### Recommendation

We recommend using `keccak256("ALL_LIMITS_MANAGER_ROLE")` for the sake of consistency.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/e2fad5eacd1bd6778d651e66c9864f593b851595

## `_checkAnnualBalancesIncrease()` doesn't include withdrawn ether

### Description

At the line [OracleReportSanityChecker.sol#L422](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol#L422), `_checkAnnualBalancesIncrease()` doesn't include withdrawn ether like `_checkOneOffCLBalanceDecrease()`.

Consider a case:

```solidity
// Initial state
preCLBalance = x;
withdrawalVaultBalance = 0;

// The validators receive 16 Ether rewards
// No validators exited or deposited
// During the next oracle report call:
preCLBalance = x;
withdrawalVaultBalance = 0;
postCLBalance = x + 16;

// the check is made for 16 Ether

// The validators receive 16 Ether rewards
// 16 Ether is automatically skimmed to withdrawal vault
// No validators exited or deposited
// During the next oracle report call:
preCLBalance = x + 16;
withdrawalVaultBalance = 16;
postCLBalance = x + 16;

// there is no check even though CL balance increased by 16 Ether

// The validators receive 16 Ether rewards
// No Ether is automatically skimmed to withdrawal vault
// 1 validator exited, no validators deposited
// During the next oracle report call:
preCLBalance = x + 16;
withdrawalVaultBalance = 48;
postCLBalance = x + 16 + 16 - 32 = x;

// there is no check even though CL balance increased by 16 Ether
```

Due to partial rewards skimming and validator exits there is inconsistency when the check is applied. One approach would be to save the withdrawal vault balance at the end of the previous report and during the next report add to `postCLBalance` only the ether which arrived between the reports. In that case, for the above example, every check would be for 16 ether.

### Recommendation

We recommend checking if this is intended behavior.

### Status

Acknowledged

### Client's comments

> The current behavior is intended. Withdrawals vault balance has its check separate from the annual CL balance increase. Counting it in the CL balance check introduces the risk of DOS via ETH sending into WithdrawalsValue

## SanityChecker includes withdrawn ether from past reports

### Description

The function [`OracleReportSanityChecker._checkOneOffCLBalanceDecrease()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol#L548) checks if the CL balance can only decrease by a limited amount since the last report. The variable `_unifiedPostCLBalance` includes withdrawal vault balance to account for withdrawals that have happened since the last oracle report. However, it also includes withdrawn ether from validators which exited before the latest oracle report. This can lead to a situation where when the CL balance does decrease, the check passes when it should not pass.

Consider a case:

```solidity
// Initial state with one validator and some ether from withdrawals and partial rewards
preCLBalance = 32;
withdrawalVaultBalance = 16;

// The validator is slashed/penalized for 16 Ether
// During oracle report call:
preCLBalance = 32;
withdrawalVaultBalance = 16;
postCLBalance = 16;
unifiedPostCLBalance = 32;

// _checkOneOffCLBalanceDecrease() passes since preCLBalance == unifiedPostCLBalance
// even though CL balance decreased
```

One approach would be to save the withdrawal vault balance at the end of the previous report and during the next report add to `postCLBalance` only the ether which arrived between the reports.

### Recommendation

We recommend checking if this is intended behavior.

### Status

Acknowledged

### Client's comments

> The current behavior is intended. The withdrawal vault balance is zeroed most of the time (with an exception of positive rebase smoothening, however, the withdrawals vault balance is applied first for the rebase limiter).

## Incorrect check

### Description

In the function [`SigningKeys.saveKeysSigs()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/lib/SigningKeys.sol#L44), the sum `_startIndex + _keysCount` can be `UINT64_MAX + 1`. In that case the function will return new total keys count `UINT64_MAX + 1`, but total keys count should be limited to `UINT64_MAX`.

### Recommendation

We recommend removing `- 1` at the line [SigningKeys.sol#L44](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/lib/SigningKeys.sol#L44).

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/f7680b177516e1df4302fffc566744a19c7414bc

## No zero check for `_maxPositiveTokenRebase`

### Description

There is no zero value check for parameter `_maxPositiveTokenRebase` in the function [`setMaxPositiveTokenRebase()`](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol#L297). The comment says that passing zero value is prohibited.

### Recommendation

We recommend adding a zero value check.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/df51ab08ce94302b71c4f699022eed2c99e1aac2

## Unsafe casting

### Description

There is unsafe casting from `uint64` to `int64` at the lines:
- [NodeOperatorsRegistry.sol#L566](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L566)
- [NodeOperatorsRegistry.sol#L801](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L801)

There is unsafe casting from `int64` to `uint64` at the lines:
- [NodeOperatorsRegistry.sol#L582](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L582)
- [NodeOperatorsRegistry.sol#L672](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L672)

### Recommendation

We recommend using safe casting.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/b3eb257b29ae56f120ac6a3696eea05d65e29e78

## Unsafe math

### Description

There is unsafe math at the lines:
- [NodeOperatorsRegistry.sol#L582](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L582)
- [NodeOperatorsRegistry.sol#L620](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L620)
- [NodeOperatorsRegistry.sol#L672](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L672)
- [NodeOperatorsRegistry.sol#L706](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L706)
- [NodeOperatorsRegistry.sol#L725](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L725)
- [NodeOperatorsRegistry.sol#L828-L829](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L828-L829)
- [NodeOperatorsRegistry.sol#L867](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L867)
- [NodeOperatorsRegistry.sol#L946](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L946)
- [NodeOperatorsRegistry.sol#L1109](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L1109)
- [NodeOperatorsRegistry.sol#L1168](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L1168)
- [NodeOperatorsRegistry.sol#L1207](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L1207)

### Recommendation

We recommend using safe math for sanity checks.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/426dbc73b796075ea5f0e620b8856370e823e8ce, https://github.com/lidofinance/lido-dao/pull/719/commits/b79476b4a427ca4b14d03a55ae834f8198640d76

## Unused function parameters

### Description

In the function [`BaseOracle._handleConsensusReport()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/oracle/BaseOracle.sol#L232-L233), the parameters `report` and `prevSubmittedRefSlot` are unused in `AccountingOracle.sol` and `ValidatorsExitBusOracle.sol`.

### Recommendation

We recommend removing unused parameters.

### Status

Acknowledged

### Client's comments

> We’ll consider removing these unused parameters in a future upgrade.

## Unused return value

### Description

The function [`ValidatorsExitBusOracle._processExitRequestsList()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/oracle/ValidatorsExitBusOracle.sol#L362) returns value `uint256`, but the return value is unused at the line [ValidatorsExitBusOracle.sol#L344](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/oracle/ValidatorsExitBusOracle.sol#L344).

### Recommendation

We recommend removing the return value.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/7a93c4a7c71e55783e7917871d94fef275be2bdf

## No checks for node operator id and validator index

### Description

In the function [`ValidatorsExitBusOracle._processExitRequestsList()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/oracle/ValidatorsExitBusOracle.sol#L399-L400), there are no checks if the node operator exists `nodeOpId < IStakingModule.getNodeOperatorsCount()`, and if the node operator validator index is valid `valIndex < totalDepositedValidators`.

### Recommendation

We recommend considering adding sanity checks for these values.

### Status

Acknowledged

### Client's comments

> Checking for a validator index is impossible since the total number of deposited validators is currently unavailable on the execution layer, so the only way to check it is to require the oracle to provide it which would make the check useless.
> We’ll consider adding the check for a valid node operator id in the future. That said, even if the oracle submits a non-existent node operator id, this won’t lead to any immediate incorrectness since exit requests are events interpreted offchain by affected node operators, and in the case of a non-existent node operator id there would just be no one to interpret the erroneous event.
> However, if an operator with this id is added later, its last requested validator index would be initially set to an incorrect value instead of zero, and recovering from this would require upgrading the contract—so the proposed check is still desired.

## Missing zero address check

### Description

In the constructor at the line [AccountingOracle.sol#L157](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/oracle/AccountingOracle.sol#L157), there is no check if address `lido` is not `address(0)`.

At the lines [OracleReportSanityChecker.sol#L149-L150](https://github.com/lidofinance/lido-dao/blob/2bce10d4f0cb10cde11bead4719a5bcde76b93f9/contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol#L149-L150), there are no zero address checks for parameters `_lidoLocator` and `_admin`.

### Recommendation

We recommend to add zero address checks.

### Status

DOUBLE

## No check for `_etherToLockOnWithdrawalQueue`

### Description

In the function [`_collectRewardsAndProcessWithdrawals()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.4.24/Lido.sol#L845-L848), there is no check if `_etherToLockOnWithdrawalQueue <= address(this).balance`. The oracle can send the `_lastFinalizableRequestId` parameter such that there is not enough ether in the `Lido` contract balance to finalize withdrawals.

### Recommendation

We recommend checking if `_etherToLockOnWithdrawalQueue <= address(this).balance`.

### Status

Acknowledged

### Client's comments

> The call, anyway, will be reverted in an attempt to transfer the ether to the withdrawal queue.

## Exited validators count might be bigger than total deposited

### Description

In the function [`StakingRouter.updateExitedValidatorsCountByStakingModule()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L271), it is possible to set `_exitedValidatorsCounts[i]` to be bigger than `totalDepositedValidators`. In that case, the oracle report will revert due to underflow at the line [StakingRouter.sol#L1088-L1090](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L1088-L1090).

### Recommendation

We recommend adding a check to avoid unexpected reverts.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/8a9bd670ec772d1c441b72a47101dcc3ee8bc9a7

## Missing checks for `_requestId` out of range

### Description

In the functions [`WithdrawalQueueBase.findCheckpointHint()`](https://github.com/lidofinance/lido-dao/blob/89ad4cc35407609081afb94df535d71b27bb44b7/contracts/0.8.9/WithdrawalQueueBase.sol#L197) and [`WithdrawalQueueBase._claimWithdrawalTo()`](https://github.com/lidofinance/lido-dao/blob/89ad4cc35407609081afb94df535d71b27bb44b7/contracts/0.8.9/WithdrawalQueueBase.sol#L434). If `_requestId > getLastRequestId()`, the functions revert with `RequestNotFinalized()` even though the request doesn't exist.

### Recommendation

We recommend checking if `_requestId > getLastRequestId()` and revert with `InvalidRequestId()` like in the function [`WithdrawalQueueBase.getWithdrawalRequestStatus()`](https://github.com/lidofinance/lido-dao/blob/89ad4cc35407609081afb94df535d71b27bb44b7/contracts/0.8.9/WithdrawalQueueBase.sol#L159).

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/58e160157264a7ccef2bafaf95a2218845914ea7

## Unused errors

### Description

The errors [`Burner.NotEnoughExcessStETH()`](https://github.com/lidofinance/lido-dao/blob/89ad4cc35407609081afb94df535d71b27bb44b7/contracts/0.8.9/Burner.sol#L60), [`WithdrawalQueue.Unimplemented()`](https://github.com/lidofinance/lido-dao/blob/89ad4cc35407609081afb94df535d71b27bb44b7/contracts/0.8.9/WithdrawalQueue.sol#L70), [`WithdrawalQueue.LengthsMismatch()`](https://github.com/lidofinance/lido-dao/blob/89ad4cc35407609081afb94df535d71b27bb44b7/contracts/0.8.9/WithdrawalQueue.sol#L74) are unused.

### Recommendation

We recommend removing unused errors.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/97f11205c69bc1a655a139f0de4dd5b5e36ab2df

## Oracle can finalize withdrawals in bunker mode

### Description

In the function [`Lido._handleOracleReport()`](https://github.com/lidofinance/lido-dao/blob/89ad4cc35407609081afb94df535d71b27bb44b7/contracts/0.4.24/Lido.sol#L1241), Lido relies on the oracle to not finalize withdrawals when in bunker mode. There is no check if the bunker mode is active in `WithdrawalQueue`. In the bunker mode, withdrawals should not be finalized since the penalties will be socialized only between holders making it beneficial to withdraw.

### Recommendation

We recommend to add a sanity check if the bunker mode is active.

### Status

DOUBLE

## Unclear comments

### Description

At the lines:
- [OracleReportSanityChecker.sol#L446](https://github.com/lidofinance/lido-dao/blob/89ad4cc35407609081afb94df535d71b27bb44b7/contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol#L446), assigned to burn or already burnt?

### Recommendation

We recommend checking if the comments are correct.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/f11eb9ae7b2f044daebdca23fc974159c3cf2e61, https://github.com/lidofinance/lido-dao/commit/55c2b09f2a9e263666bae04c9e82c210477f71c1

## Duplicate code

### Description

At the lines:

- [StakingRouter.sol#L234-L235](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L234-L235), it is possible to use the function [`_getStakingModuleById()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L1135)
- It is possible to use the function [`_getStakingModuleAddressById()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L1144) in the following:
  - [StakingRouter.sol#L255](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L255), 
  - [StakingRouter.sol#L265](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L265), 
  - [StakingRouter.sol#L313](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L313), 
  - [StakingRouter.sol#L559-L560](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L559-L560), 
  - [StakingRouter.sol#L577-L578](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L577-L578),
- [StakingRouter.sol#L426-L427](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L426-L427), it is possible to use [`_getStakingModuleAddressByIndex()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L1148)

### Recommendation

We recommend to use these functions to avoid duplicate code.

### Status

DOUBLE

## Gas optimization

### Description

At the line [StakingRouter.sol#L295](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L295), it is possible to use `_stakingModuleIds[i]` instead of `stakingModule.id` which reads from storage.

### Recommendation

We recommend using `_stakingModuleIds[i]`.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/64765b2f0f2c45b52438ac416784101adb3b59ce

## Missing zero check when reporting minted rewards

### Description

In the function [`reportRewardsMinted()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/StakingRouter.sol#L266), `_totalShares[i]` can be `0` if the staking module is stopped and no shares were minted for it.
The function [`onRewardsMinted()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.8.9/interfaces/IStakingModule.sol#L80-L82) should be called if the rewards were minted for the module.

### Recommendation

We recommend adding a zero check for `_totalShares[i]`.

### Status

Fixed at https://github.com/lidofinance/lido-dao/pull/656/commits/b86c4697bccf5b6b9cf5cc1a36ce682734e50707

### Client's comments

> Since the function doesn’t mark the memory past the new array data end as free (e.g. by decreasing the free memory pointer) it doesn’t affect the correctness of any code respecting the Solidity memory safety rules as such code should never assume that any memory prior to the slot the free memory pointer points to is unused or zeroed out. Zeroing the memory wouldn’t hurt but will consume additional gas, and saving gas is the main purpose of this function.

## Possible underflow in `trimUint256Array()`

### Description

In the function [`MemUtils.trimUint256Array()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/common/lib/MemUtils.sol#L86), there is no check if `_trimBy` is not bigger than `_arr.length`.

### Recommendation

We recommend adding a check to avoid possible underflow.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/893d0a1c79a14680a0062e2d138f3d89f4004264

### Client's comments

> The function is not used anymore and was completely removed

## Typo in comment

### Description

At the line [NodeOperatorsRegistry.sol#L325](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L325), it should be `activate` instead of `deactivate`.

### Recommendation

We recommend fixing the typo.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/14c355beb17bf430687185af1526113addba72ab

## The function `updateTargetValidatorsLimits()` is not called by staking router

### Description

The function [`updateTargetValidatorsLimits()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L593-L595) is access restricted to `STAKING_ROUTER_ROLE`, but the staking router doesn't call this function.

### Recommendation

We recommend to check if the correct role is used.

### Status

DOUBLE

### Client's comments

> Such functionality was implemented intentionally, to allow node operators to fix problems with keys (for example when the node operator loses control over signing keys) even while it is deactivated.

## Typo in `_addSigningKeys()`

### Description

At the line [NodeOperatorsRegistry.sol#L997](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.4.24/nos/NodeOperatorsRegistry.sol#L997), it should be `SUMMARY_TOTAL_KEYS_COUNT_OFFSET` instead of `TOTAL_KEYS_COUNT_OFFSET`.

### Recommendation

We recommend fixing the typo.

### Status

Fixed at https://github.com/lidofinance/lido-dao/commit/8bb30e04f98f4cc539ea624d177704bcef3d96b5

## Missing length check in `loadKeysSigs()`

### Description

In the function [`loadKeysSigs()`](https://github.com/lidofinance/lido-dao/blob/e57517730c3e11a41e9cbc32ce018726722335b7/contracts/0.4.24/lib/SigningKeys.sol#L149), there is no length check for `_pubkeys` and `_signatures`. These variables should be of length to fit `_keysCount` number of keys and signatures respectively starting from `_bufOffset`.

### Recommendation

We recommend adding a sanity check to ensure the loaded keys fit.

### Status

Acknowledged

### Client's comments

> This check was skipped for gas savings because loadKeysSigs is used in pair with initKeysSigsBuf which reserves correct space for keys and signatures.

### Client's comment

> From the perspective of NodeOperatorsRegistry, an empty report is considered valid.

### Client's comment

> Method StakingRouter.getNodeOperatorDigest() is designed for off-chain calls where gas costs are not particularly important. In turn, NodeOperatorsRegistry.getNodeOperatorSummary() is supposed to be called by the onchain consumer (currently is called by StakingRouter.unsafeSetExitedValidatorsCount()), and adding isActive to the result will add additional SLOAD to the call.


### Client's comment

> It’s a design decision. The Lido contract always has to have this role, but NodeOperatorsRegistry might be deprived of this role in the future. Because of that, REQUEST_BURN_SHARES_ROLE is granted to the NodeOperatorsRegistry on the standalone step of the deployment.


## No checks for a different values in the setter functions

### Description

In the contract `DepositSecurityModule`, the functions [`_setOwner()`](https://github.com/lidofinance/lido-dao/blob/feafec437669a131a9e3c33ca680618d490c4fef/contracts/0.8.9/DepositSecurityModule.sol#L144), [`_setPauseIntentValidityPeriodBlocks()`](https://github.com/lidofinance/lido-dao/blob/feafec437669a131a9e3c33ca680618d490c4fef/contracts/0.8.9/DepositSecurityModule.sol#L164), [`_setMaxDeposits()`](https://github.com/lidofinance/lido-dao/blob/feafec437669a131a9e3c33ca680618d490c4fef/contracts/0.8.9/DepositSecurityModule.sol#L184), there are no checks if the new value is different than the current value. In the functions [`_setMinDepositBlockDistance()`](https://github.com/lidofinance/lido-dao/blob/feafec437669a131a9e3c33ca680618d490c4fef/contracts/0.8.9/DepositSecurityModule.sol#L203), [`_setGuardianQuorum()`](https://github.com/lidofinance/lido-dao/blob/feafec437669a131a9e3c33ca680618d490c4fef/contracts/0.8.9/DepositSecurityModule.sol#L222) there are checks if the new value is different.

### Recommendation

We recommend adding checks in the functions [`_setOwner()`](https://github.com/lidofinance/lido-dao/blob/feafec437669a131a9e3c33ca680618d490c4fef/contracts/0.8.9/DepositSecurityModule.sol#L144), [`_setPauseIntentValidityPeriodBlocks()`](https://github.com/lidofinance/lido-dao/blob/feafec437669a131a9e3c33ca680618d490c4fef/contracts/0.8.9/DepositSecurityModule.sol#L164), [`_setMaxDeposits()`](https://github.com/lidofinance/lido-dao/blob/feafec437669a131a9e3c33ca680618d490c4fef/contracts/0.8.9/DepositSecurityModule.sol#L184).

### Status

Acknowledged.

### Client's comment

> For code simplicity, checks on setting the same values were omitted. These methods are supposed to be called rarely, so gas extra costs are not so important in this case.

## StakingRouter can return exited validators that hasn't been updated

### Description

The function [`StakingRouter.getStakingModuleSummary()`](https://github.com/lidofinance/lido-dao/blob/feafec437669a131a9e3c33ca680618d490c4fef/contracts/0.8.9/StakingRouter.sol#L701) returns `summary.totalExitedValidators` taken from staking module which can differ from the total exited stored in the staking router.

### Recommendation

We recommend returning the maximum of both values.

### Status

Acknowledged.

### Client's comment

> This method is supposed to return the same values as IStakingModule.getStakingModuleSummary().

## Possible risk of DOS via ETH sending into Consensus Layer

### Description

The function [`OracleReportSanityChecker._checkAnnualBalancesIncrease()`](https://github.com/lidofinance/lido-dao/blob/feafec437669a131a9e3c33ca680618d490c4fef/contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol#L564) checks that the consensus layer balance of Lido validators doesn't increase beyond a certain annual limit. This can lead to DOS attack to block oracle reports. An attacker can send a deposit to the Ethereum 2.0 deposit contract with a public key of a Lido validator to increase that validator's balance on the consensus layer such that `annualBalanceIncrease` is bigger than the limit. The function `apply_deposit()` from [https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#deposits](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#deposits):

```python
def apply_deposit(state: BeaconState,
                  pubkey: BLSPubkey,
                  withdrawal_credentials: Bytes32,
                  amount: uint64,
                  signature: BLSSignature) -> None:
    validator_pubkeys = [v.pubkey for v in state.validators]
    if pubkey not in validator_pubkeys:
        # Verify the deposit signature (proof of possession) which is not checked by the deposit contract
        deposit_message = DepositMessage(
            pubkey=pubkey,
            withdrawal_credentials=withdrawal_credentials,
            amount=amount,
        )
        domain = compute_domain(DOMAIN_DEPOSIT)  # Fork-agnostic domain since deposits are valid across forks
        ...

        # Add validator and balance entries
        state.validators.append(get_validator_from_deposit(pubkey, withdrawal_credentials, amount))
        state.balances.append(amount)
    else:
        # Increase balance by deposit amount
        index = ValidatorIndex(validator_pubkeys.index(pubkey))
        increase_balance(state, index, amount)
```

The attacker can send a deposit message without having the Lido validator's private key and increase the balance.

The accounting oracle calculates the CL balance as the sum of current balances [https://github.com/lidofinance/lido-oracle/blob/7dd48b559611baa2237aa5cab212647408aae867/src/modules/accounting/accounting.py#L204](https://github.com/lidofinance/lido-oracle/blob/7dd48b559611baa2237aa5cab212647408aae867/src/modules/accounting/accounting.py#L204):

```python
def _get_consensus_lido_state(self, blockstamp: ReferenceBlockStamp) -> tuple[int, Gwei]:
    lido_validators = self.w3.lido_validators.get_lido_validators(blockstamp)

    count = len(lido_validators)
    total_balance = Gwei(sum(int(validator.balance) for validator in lido_validators))

    logger.info({'msg': 'Calculate consensus lido state.', 'value': (count, total_balance)})
    return count, total_balance
```

However, this attack is unlikely to have financial incentive for the attacker. Depending on the `annualBalanceIncreaseBPLimit` it will be increasingly expensive to block oracle reports for a long time.

### Recommendation

We recommend assessing the risk of such an attack.

### Status

Acknowledged.

### Client's comment

> Considering the current TVL of the protocol and the expected value for annualBalanceIncreaseBPLimit equal to 10.00%, the attack will be insanely expensive for an attacker to DOS multiple reports in a row still with a tolerable impact on the protocol. Moreover, Lido DAO can raise the annualBalanceIncreaseBPLimit in case of such an attack.
