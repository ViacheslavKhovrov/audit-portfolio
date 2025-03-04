# CRITICAL

## veYFI balance might not decrease

### Description

It is possible for a user to create a lock longer than `MAX_LOCK_DURATION`, but such that `kink.ts` at the line: https://github.com/yearn/veYFI/blob/696bb76be86a601f25cda577bfb9dc14daa91079/contracts/VotingYFI.vy#L136 is less than `block.timestamp` since it rounds down to a week. Then in the function `balanceOf()`, function `replay_slope_changes()` will not consider the change of slope at `kink.ts` resulting in a balance of user that doesn't decrease with time.

### Recommendation

It is recommended to refactor how timestamps are rounded to weeks and check if all slope changes are applied correctly.

### Status
Fixed at https://github.com/yearn/veYFI/pull/179

## Incorrect balance calculation leading to higher reward

### Description

At the line: https://github.com/yearn/veYFI/blob/696bb76be86a601f25cda577bfb9dc14daa91079/contracts/RewardPool.vy#L266.

The balance calculation can be incorrect if a user creates a lock longer than 4 years such that checkpoint slope is `0`. Then `balance_of` would be the initial veYFI balance at `old_user_point.ts` without factoring change of slope in kink. Then a user might get a bigger reward than they were supposed to.

### Recommendation

It is recommended to use function `balanceOf` from the `VotingYFI` contract.

### Status
Fixed at https://github.com/yearn/veYFI/pull/182

# HIGH

## Incorrect interface

### Description
At the lines: https://github.com/yearn/veYFI/blob/696bb76be86a601f25cda577bfb9dc14daa91079/contracts/RewardPool.vy#L9-L16.

The interface for the `VotingYFI` contract is incorrect.

### Recommendation

It is recommended to update the interface and the function calls to that contract.

### Status
Fixed at https://github.com/yearn/veYFI/commit/4e9dfea57973eb2116e2691ae7f8e640ff49597d

## Incorrect balance calculation

### Description
At the line: https://github.com/yearn/veYFI/blob/696bb76be86a601f25cda577bfb9dc14daa91079/contracts/RewardPool.vy#L175.

The function `ve_for_at()` returns the veYFI balance for a user at the timestamp. But the calculation can be incorrect if a user creates a lock longer than 4 years such that the checkpoint slope is `0`. Then the balance at `_timestamp` would return the initial veYFI balance without factoring change of slope in kink.

### Recommendation

It is recommended to use function `balanceOf` from the `VotingYFI` contract.

### Status
Fixed at https://github.com/yearn/veYFI/pull/182

# MEDIUM

## Tokens may be lost if `checkpoint` functions are not called for 20 weeks

### Description

If the function `_checkpoint_token()` is not called for more than 20 weeks, then `since_last` at the line https://github.com/yearn/veYFI/blob/696bb76be86a601f25cda577bfb9dc14daa91079/contracts/RewardPool.vy#L92 will be bigger than 20 weeks, but the `for` loops only adds rewards for 20 weeks, meaning some tokens will be lost.

If the function `_checkpoint_total_supply()` is not called for more than 20 weeks, then when claiming rewards `self.ve_supply[week_cursor]` will reach `0` reverting the transaction.

### Recommendation

It is recommended to refactor how rewards are distributed in edge cases and increase weeks in `_checkpoint_total_supply()`.

### Status
Fixed at https://github.com/yearn/veYFI/commit/cfdf194a1a80a6ffffd2ce30c5e6d6127f8fb6d2

## Incorrect indentation

### Description
https://github.com/yearn/veYFI/blob/21d85c7eef51f380e33f217bbb6e8f89cad759a1/contracts/RewardPool.vy#L211
With incorrect indentation the contract won't compile.

### Recommendation
It is recommended to correct indentation.

### Status
Fixed at https://github.com/yearn/veYFI/commit/961ef6c03595a205336e5120d68af682dd22e03e

# INFORMATIONAL

## No check for address zero

### Description
At the lines: https://github.com/yearn/veYFI/blob/696bb76be86a601f25cda577bfb9dc14daa91079/contracts/VotingYFI.vy#L87-L88.

There is no check for address zero for parameters `token`, `reward_pool`.

### Recommendation

It is recommended to add sanity checks.

### Status
Acknowledged

### Client's comments
> yearn thinks this issue can be mitigated during proper ops deployment of contract risk and impact are low since if addresses are not configured contract can be redeployed

## Unnecessary condition

### Description
At the line: https://github.com/yearn/veYFI/blob/696bb76be86a601f25cda577bfb9dc14daa91079/contracts/RewardPool.vy#L100.

The check `block.timestamp == t` is unnecessary since if `since_last == 0`, then `t` is equal to `block.timestamp`.

### Recommendation
It is recommended to remove this condition.

### Status
Acknowledged

## Unreachable code

### Description
At the line: https://github.com/yearn/veYFI/blob/696bb76be86a601f25cda577bfb9dc14daa91079/contracts/RewardPool.vy#L106.

The condition is never true since `next_week` can never be equal to `t`. If `since_last == 0`, then `block.timestamp < next_week`.

### Recommendation
It is recommended to remove the if condition.

### Status
Acknowledged

## Unnecessary gas usage

### Description
At the line: https://github.com/yearn/veYFI/blob/696bb76be86a601f25cda577bfb9dc14daa91079/contracts/RewardPool.vy#L86.

In the function `_checkpoint_token()`, if `to_distribute` is `0`, then the `for` loop at line https://github.com/yearn/veYFI/blob/696bb76be86a601f25cda577bfb9dc14daa91079/contracts/RewardPool.vy#L97 would not change anything.

### Recommendation
It is recommended to check if `to_distribute` is `0` before the loop.

### Status
Fixed at https://github.com/yearn/veYFI/pull/193

## Unnecessary check

### Description
https://github.com/yearn/veYFI/blob/21d85c7eef51f380e33f217bbb6e8f89cad759a1/contracts/RewardPool.vy#L212
The uint256 `balance_of` var is already compared to zero, so it isn't necessary to check if `balance_of > 0`.

### Recommendation
It is recommended to remove the check.

### Status
Fixed at https://github.com/yearn/veYFI/commit/fc353687ff72fe20426b1f16ab61e9ba443401ae
