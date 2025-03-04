# CRITICAL

Not found

# HIGH

Not found

# MEDIUM

Not found

# INFORMATIONAL

## Single point of failure in `transferOwnership()`

### Description

At the line https://github.com/lidofinance/insurance-fund/blob/e2aadf7b548a69860a3f535faaf7170712466463/contracts/InsuranceFund.sol#L14.

Currently, `transferOwnership()` sets `_owner` if called with a non-zero address which is a potential risk if the address is not controlled by the Lido DAO and could lead to the loss of assets in the contract.

It is recommended to use a 2-step transfer of ownership where the pending owner has to accept ownership to become the owner. 

For reference, OpenZeppelin's [Ownable2Step.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol) implementation.

### Recommendation

Consider using a 2-step transfer of ownership.

### Status
Acknowledged

### Client's comments
> While we understand the benefit of the recommended ownership model, we will be proceeding with the original ownership contract. We do not plan to use the transfer ownership feature often (if ever), so the suggested model does not present any real benefits for us.