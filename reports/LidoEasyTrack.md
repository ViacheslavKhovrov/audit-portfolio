# CRITICAL

Not found

# HIGH

## Possible griefing attack to cancel motions

### Description

At the line [EasyTrack.sol#L146](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EasyTrack.sol#L146), the motion snapshot block is set as the current `block.number`. An attacker can backrun the create motion transaction, take a flashloan of the governance token and object the motion multiple times from different addresses until the objections threshold is reached. 

### Recommendation

It is recommended to set the snapshot block to `block.number - 1`.

### Status
Acknowledged

### Client's comments
> We accept this risk, because the costs of redeployment of EasyTrack is higher than the possible benefits. 
Now we mitigate the risk as follows:
> - Have monitoring for unusual on-chain activities (single-block objections occurred within the block of the motion creation)
> - Once someone decides to exploit the behavior, the possibility of front-running could be circumvented by using the private communication channels with block proposers (e.g., Flashbots) to prevent tx interception and front-running inside the mempool.
> - Full-fledged Aragon voting can still perform the necessary actions
> - We created an issue and PR for this improvement:
https://github.com/lidofinance/easy-track/issues/26
https://github.com/lidofinance/easy-track/pull/25

# MEDIUM

Not found

# INFORMATIONAL

## Inconsistent naming of function parameters

### Description

At the lines:
- [AllowedRecipientsRegistry.sol#L143](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/AllowedRecipientsRegistry.sol#L143)
- [AllowedRecipientsRegistry.sol#L130](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/AllowedRecipientsRegistry.sol#L130)
- [RewardProgramsRegistry.sol#L115](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/RewardProgramsRegistry.sol#L115)

It is recommended to rename the function parameters `_evmScriptFactory` and `_address` to improve readability.

### Recommendation

Consider renaming these function parameters to be more in line with the context of each contract.

### Status
Acknowledged

### Client's comments
> Fixed for AllowedRecipientsRegistry.sol
We plan to replace the deployed reward program factories with new corresponding factories for allowed recipients. Reward factories will never deploy again

## Variables can be `immutable`

### Description

At the lines:
- [TopUpAllowedRecipients.sol#L35](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/TopUpAllowedRecipients.sol#L35)
- [TopUpAllowedRecipients.sol#L38](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/TopUpAllowedRecipients.sol#L38)
- [AddAllowedRecipient.sol#L25](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/AddAllowedRecipient.sol#L25)
- [RemoveAllowedRecipient.sol#L24](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/RemoveAllowedRecipient.sol#L24)

The variables `token` and `allowedRecipientsRegistry` can be declared as `immutable` since they are set in the constructor and do not change.

### Recommendation

It is recommended to declare these variables as `immutable`.

### Status
Acknowledged

### Client's comments
> We deliberately do not use immutable variables for the `token` and `AllowedRecipientsRegistry` for the following reason: we plan to deploy a set of factories for allowed recipients many times for different committees. To save time on deployment and verification, and for security purposes, we want the contract bytecode be the same for different `token` and `allowedRecipientsRegistry` values.

## No checks for address zero

### Description

At the line [AddAllowedRecipient.sol#L51](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/AddAllowedRecipient.sol#L51), the `recipientAddress` is not checked for `address(0)`.

At the line [AddRewardProgram.sol#L50](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/AddRewardProgram.sol#L50), the `rewardProgramAddress` is not checked for `address(0)`.

At the line [EVMScriptFactoriesRegistry.sol#L57](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactoriesRegistry.sol#L57), the `_evmScriptFactory` is not checked for `address(0)`.

At the line [EasyTrack.sol#L258](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EasyTrack.sol#L258), the `_evmScriptExecutor` is not checked for `address(0)`.

### Recommendation

It is recommended to ensure these variables are not `address(0)`.

### Status
Acknowledged

### Client's comments
> Added check for AddAllowedRecipient.sol.
Other contracts will not be redeployed anytime soon

## Incorrect NatSpec comments

### Description

At the line [AddAllowedRecipient.sol#L43](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/AddAllowedRecipient.sol#L43), the encoded tuple should be `(address recipientAddress, string title)`.

At the line [AddRewardProgram.sol#L42](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/AddRewardProgram.sol#L42), the encoded tuple should be `(address _rewardProgram, string _title)`.

### Recommendation

It is recommended to fix these comments.

### Status
Acknowledged

### Client's comments
> Fixed for AddAllowedRecipient.sol

## Possible to create a motion with transfer amount bigger than single transfer limit

### Description

At the lines:
- [TopUpAllowedRecipients.sol#L134](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/TopUpAllowedRecipients.sol#L134)
- [TopUpLegoProgram.sol#L107](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/TopUpLegoProgram.sol#L107)
- [TopUpRewardPrograms.sol#L114](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/TopUpRewardPrograms.sol#L114)

The variables `_amounts[i]` are not checked for max limit, so it is possible to create a motion with transfer amount bigger than single transfer limit implemented in [LIP-13](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-13.md). If such a motion is created it has to be cancelled since it cannot be enacted.

### Recommendation

Consider not allowing to create such motions.

### Status
Acknowledged

### Client's comments
> We accept this risk, there is no possibility of losing funds. The check implemented in LIP-13 is a safety check and should not be part of the logic for EasyTrack factories

## Event parameter can be `indexed`

### Description

At the line [EasyTrack.sol#L39](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EasyTrack.sol#L39), the event parameter `_creator` can be set as `indexed`.

### Recommendation

Consider setting the parameter `_creator` as `indexed`.

### Status
Acknowledged    

### Client's comments
> while the cost of EasyTrack redeploy is high, we will postpone these improvements

## Gas optimization by replacing `memory` with `calldata`

### Description

Files:
- [contracts/EVMScriptFactories/AddAllowedRecipient.sol](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/AddAllowedRecipient.sol)
- [contracts/EVMScriptFactories/AddRewardProgram.sol](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/AddRewardProgram.sol)
- [contracts/EVMScriptFactories/IncreaseNodeOperatorStakingLimit.sol](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/IncreaseNodeOperatorStakingLimit.sol)
- [contracts/EVMScriptFactories/RemoveAllowedRecipient.sol](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/RemoveAllowedRecipient.sol)
- [contracts/EVMScriptFactories/RemoveRewardProgram.sol](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/RemoveRewardProgram.sol)
- [contracts/EVMScriptFactories/TopUpAllowedRecipients.sol](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/TopUpAllowedRecipients.sol)
- [contracts/EVMScriptFactories/TopUpLegoProgram.sol](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/TopUpLegoProgram.sol)
- [contracts/EVMScriptFactories/TopUpRewardPrograms.sol](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactories/TopUpRewardPrograms.sol)
- [contracts/EVMScriptExecutor.sol](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptExecutor.sol)
- [contracts/EVMScriptFactoriesRegistry.sol](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EVMScriptFactoriesRegistry.sol)
- [contracts/EasyTrack.sol](https://github.com/lidofinance/easy-track/blob/22c955540e6b9fb5cb46b2ea40bebf367d38eb24/contracts/EasyTrack.sol)

External calls to functions with `memory` parameters can be made more gas efficient by replacing `memory` with `calldata`, as long as the memory parameters are not modified.

### Recommendation

Consider replacing `memory` with `calldata`.

### Status
Acknowledged

### Client's comments
> To implement these changes, a redeploy of EasyTrack is required. Overrun gas cost is insignificant in terms of DAO payments, and it does not add any risks.