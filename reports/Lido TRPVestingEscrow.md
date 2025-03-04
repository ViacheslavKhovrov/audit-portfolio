# CRITICAL

Not found

# HIGH

Not found

# MEDIUM

Not found

# INFORMATIONAL

## The function `claim()` doesn't return value claimed

### Description

At the line [contracts/VestingEscrow.vy#L179](https://github.com/lidofinance/lido-vesting-escrow/blob/dfe7bde8911382525819048b3beda524b2c3a3bf/contracts/VestingEscrow.vy#L179).

The function `claim()` transfers `claimable` amount of tokens to the beneficiary, but `claimable` can be smaller than `amount` parameter. If the recipient is a contract, it may be useful to return `claimable` value. 

### Recommendation

Consider returning `claimable` value in the function `claim()`.

### Status
Fixed at https://github.com/lidofinance/lido-vesting-escrow/commit/def81081543a32b1f1a4968a5215e47c5f131619

## No check if `amount` to be vested is bigger than `0`

### Description

In the function `deploy_vesting_contract()` at the line [contracts/VestingEscrowFactory.vy#L89](https://github.com/lidofinance/lido-vesting-escrow/blob/def81081543a32b1f1a4968a5215e47c5f131619/contracts/VestingEscrowFactory.vy#L89).

The function parameter `amount` is not checked to be bigger than `0`. Therefore, it is possible to create escrows with no tokens.

### Recommendation

It is recommended to check if `amount > 0`.

### Status
Fixed at https://github.com/lidofinance/lido-vesting-escrow/commit/69dd13adcd9c5a88da8c134b221209ccded04121