# CRITICAL

# HIGH

## Factory can stop creating vaults due to block gas limit

### Description

When creating a vault the `LenderVaultFactory` reads all of the registered vaults from storage and copies all of them to memory just to get its length at the line [contracts/peer-to-peer/LenderVaultFactory.sol#L30](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/LenderVaultFactory.sol#L30):

```solidity
uint256 numRegisteredVaults = IAddressRegistry(addressRegistry)
    .registeredVaults()
    .length;
```

Because of this, each vault creation will be more expensive in gas. At some point, it will be impossible to create new vaults due to the block gas limit. A malicious user can spam create many vaults blocking others from creating vaults.

### Recommendation

We recommend adding a function to `AddressRegistry` which returns the number of registered vaults.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/45f3eda43f6b9adc76ee59828e48f4b6fe4af212>

# MEDIUM

## Conversion and repayment periods can overlap

### Description

In the function `_repaymentScheduleCheck()` [contracts/peer-to-pool/LoanProposalImpl.sol#L615](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/LoanProposalImpl.sol#L615), the time between due timestamps has to be only bigger than `MIN_TIME_BETWEEN_DUE_DATES`. However, `conversion grace period + repayment grace period` can be bigger than `MIN_TIME_BETWEEN_DUE_DATES` leading to overlapping of conversion grace periods. Consider an example:

```solidity
conversion grace period = 1 day
repayment grace period = 29 days
time between due dates = 7 days

first due date = 0
first conversion period = [0, 1 day)
first repayment period = [1 day, 30 days)

second due date = 7 days
second conversion period = [7 days, 8 days)
second repayment period = [8 days, 37 days)
```

The repayment index increases only after repayment, so the lenders will not be able to convert after the second due date if the borrower makes the first repayment after `8 days`.

### Recommendation

We recommend ensuring that the time between due dates is bigger than the conversion grace period and repayment grace period.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/43355d3ba099b04ad632f55459578b02982fb72e>

## `_repaymentScheduleCheck()` does not take into account unsubscribe period

### Description

In the function `_repaymentScheduleCheck()` [contracts/peer-to-pool/LoanProposalImpl.sol#L615](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/LoanProposalImpl.sol#L615), the minimum time until the first due date is `LOAN_TERMS_UPDATE_COOL_OFF_PERIOD + LOAN_EXECUTION_GRACE_PERIOD + MIN_TIME_UNTIL_FIRST_DUE_DATE`. However, it does not take into account the unsubscribe period. After `acceptLoanTerms()` call, the borrower has to wait `unsubscribeGracePeriod` before calling `finalizeLoanTermsAndTransferColl()`. The `unsubscribeGracePeriod` can be unlimited, so in some scenarios the `unsubscribeGracePeriod` can overlap with the first due date, making it impossible to call `finalizeLoanTermsAndTransferColl()`.

### Recommendation

We recommend including `unsubscribeGracePeriod` in the check.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/2b51f4824c00d4ab289e10dbcda0bfc7f668ba09>

## `acceptLoanTerms()` may not be called due to an incorrect check

### Description

In the function `_repaymentScheduleCheck()` [contracts/peer-to-pool/LoanProposalImpl.sol#L615](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/LoanProposalImpl.sol#L615), the minimum time until the first due date is `LOAN_TERMS_UPDATE_COOL_OFF_PERIOD + LOAN_EXECUTION_GRACE_PERIOD + MIN_TIME_UNTIL_FIRST_DUE_DATE`. If the `proposeLoanTerms()` is called at the latest possible timestamp, `acceptLoanTerms()` cannot be called because it takes into consideration `LOAN_TERMS_UPDATE_COOL_OFF_PERIOD`. Consider an example:

```solidity
proposeLoanTerms() called at timestamp = 0
first due date = LOAN_TERMS_UPDATE_COOL_OFF_PERIOD + LOAN_EXECUTION_GRACE_PERIOD + MIN_TIME_UNTIL_FIRST_DUE_DATE
acceptLoanTerms() called at timestamp = LOAN_TERMS_UPDATE_COOL_OFF_PERIOD

acceptLoanTerms() calls _repaymentScheduleCheck() = 
LOAN_TERMS_UPDATE_COOL_OFF_PERIOD
+ LOAN_TERMS_UPDATE_COOL_OFF_PERIOD
+ LOAN_EXECUTION_GRACE_PERIOD
+ MIN_TIME_UNTIL_FIRST_DUE_DATE
> first due date
```

### Recommendation

We recommend not adding `LOAN_TERMS_UPDATE_COOL_OFF_PERIOD` when `acceptLoanTerms()` calls `_repaymentScheduleCheck()`.

### Status

Double

## `_updateSingletonState()` doesn't reset the old address

### Description

When changing the addresses of `erc721Wrapper`, `erc20Wrapper`, `mysoTokenManager` to a new address, `_updateSingletonState()` calls `_resetAddrState()` with the value of the new address instead of the old address at the line [contracts/peer-to-peer/AddressRegistry.sol#L333](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/AddressRegistry.sol#L333). Because of that `whitelistState` at the old address will still have a whitelisted state. This way states `ERC721WRAPPER`, `ERC20WRAPPER`, `MYSO_TOKEN_MANAGER` can be "occupied" by multiple addresses.

### Recommendation

We recommend using the old address in `_resetAddrState()`.

### Status

DOUBLE

## `borrowCallback()` does not support tokens with the transfer fee

### Description

The `borrowCallback()` function in the contracts `BalancerV2Looping`, `UniV3Looping` does not support tokens with transfer fee. During a borrow the `BorrowGateway` sends `initLoanAmount` of tokens. If the token has a transfer fee, the callback will not receive less than `initLoanAmount`. However, the callbacks use `initLoanAmount` for swap input amount: [contracts/peer-to-peer/callbacks/BalancerV2Looping.sol#L42](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/callbacks/BalancerV2Looping.sol#L42), [contracts/peer-to-peer/callbacks/UniV3Looping.sol#L37](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/callbacks/UniV3Looping.sol#L37), which will fail if the callback does not have enough tokens. The other contracts of the protocol make it possible to use deflationary tokens as collateral or loan tokens. Hence, the callbacks should support them.

### Recommendation

We recommend using the expected transfer fee or callback balance for swaps.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/5c5a58cc04b523aff600ab1d74de95d169b3b887>

## Wrapper approvals can be abused

### Description

In the contracts `ERC20Wrapper` and `ERC721Wrapper`, the function `createWrappedToken()` does not have access control. Hence, a malicious user can abuse user approvals to create wrapped tokens that would take all of the approved balance of users by controlling the `minter` parameter in `transferFrom()` at the lines [contracts/peer-to-peer/wrappers/ERC20/ERC20Wrapper.sol#L73](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/wrappers/ERC20/ERC20Wrapper.sol#L73), [contracts/peer-to-peer/wrappers/ERC721/ERC721Wrapper.sol#L77](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/wrappers/ERC721/ERC721Wrapper.sol#L77):

```solidity
IERC20(tokensToBeWrapped[i].tokenAddr).safeTransferFrom(
    minter,
    newErc20Addr,
    tokensToBeWrapped[i].tokenAmount
);
```

The users will unexpectedly lose their token balance and can still redeem the tokens by unwrapping them, but they will have to spend gas doing so.

### Recommendation

We recommend allowing only the `AddressRegistry` to call `createWrappedToken()`.

### Status

Double

## Borrower might not be able to repay desired amount

### Description

When borrower makes a repay, `_processRepayTransfers()` computes `reclaimCollAmount` and reverts if it is `0` at the line [contracts/peer-to-peer/BorrowerGateway.sol#L321](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/BorrowerGateway.sol#L321):

```solidity
reclaimCollAmount =
    (loan.initCollAmount * loanRepayInstructions.targetRepayAmount) /
    loan.initRepayAmount;
if (reclaimCollAmount == 0) {
    revert Errors.ReclaimAmountIsZero();
}
```

However, if the collateral is stored in a compartment, the amount the borrower reclaims is computed inside the compartment based on its balance and it can differ from the `reclaimCollAmount` computed before. Consider a simplified case:

```solidity
// let collateral be a rebasing token
// borrower makes a repay with these values
loan.initCollAmount = 1;
loanRepayInstructions.targetRepayAmount = 1;
loan.initRepayAmount = 2;
loan.amountRepaidSoFar = 0;

// in this case the call reverts since 
reclaimCollAmount = 0;

// collateral balance in the compartment increased since the beginning of the loan
collToken.balanceOf(compartment) = 2;
amount = (repayAmount * currentCompartmentBal) / repayAmountLeft = 1;

// the amount calculated in the compartment differs from reclaimCollAmount
// and the borrower should be able to make this repayment
```

### Recommendation

We recommend to compute `reclaimCollAmount` and revert if it's zero only if there is no compartment. The return value `reclaimCollAmount` is used only if there is no compartment.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/13687da3b57d9b03319dc1e6740e943d3c5835b1>

# INFORMATIONAL

## Borrower might not receive collateral for repayment

### Description

When borrower makes a repay, `_processRepayTransfers()` computes `reclaimCollAmount` and reverts if it is `0` at the line [contracts/peer-to-peer/BorrowerGateway.sol#L321](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/BorrowerGateway.sol#L321):

```solidity
reclaimCollAmount =
    (loan.initCollAmount * loanRepayInstructions.targetRepayAmount) /
    loan.initRepayAmount;
if (reclaimCollAmount == 0) {
    revert Errors.ReclaimAmountIsZero();
}
```

However, if the collateral is stored in a compartment, the amount the borrower reclaims is computed in the compartment based on its balance and it can differ from the `reclaimCollAmount` computed before. If the reclaim amount computed in the compartment is `0`, the transaction does not revert unlike above when `reclaimCollAmount == 0`.
Consider a simplified case:

```solidity
// let collateral be a rebasing token
// borrower makes a repay with these values
loan.initCollAmount = 2;
loanRepayInstructions.targetRepayAmount = 1;
loan.initRepayAmount = 2;
loan.amountRepaidSoFar = 0;

// in this case 
reclaimCollAmount = 1;

// collateral balance in the compartment decreased since the beginning of the loan
collToken.balanceOf(compartment) = 1;
amount = (repayAmount * currentCompartmentBal) / repayAmountLeft = 0;

// the amount calculated in the compartment differs from reclaimCollAmount
// and the borrower makes a repayment but does not receive any collateral
```

In this case, the `amountRepaidSoFar` will increase and the borrower can still reclaim the collateral in the future while repaying less.

### Recommendation

We recommend to revert in compartment when reclaim amount is `0` to be more consistent.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/9a4c5edc38f33bf05501d817d95ad8264e8fc217>

## Hardcoded addresses can be replaced with constant variables

### Description

At the lines [contracts/peer-to-peer/oracles/chainlink/ChainlinkBasicWithWbtc.sol#L28](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/oracles/chainlink/ChainlinkBasicWithWbtc.sol#L28), [contracts/peer-to-peer/oracles/chainlink/OlympusOracle.sol#L30](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/oracles/chainlink/OlympusOracle.sol#L30), [contracts/peer-to-peer/oracles/chainlink/UniV2Chainlink.sol#L29](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/oracles/chainlink/UniV2Chainlink.sol#L29), the hardcoded addresses can be replaced with constant variables for more clarity and consistency.

### Recommendation

We recommend to use constant variables instead of hardcoded addresses.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/91e95cd94d11c796ee3b52512964be3c321b20d5>

## New quote can be the same as the old quote in `updateOnChainQuote()`

### Description

In the function `updateOnChainQuote()` [contracts/peer-to-peer/QuoteHandler.sol#L59](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/QuoteHandler.sol#L59), there is no check if the hash of the old quote is equal to the hash of the new quote. In that case `updateOnChainQuote()` will remove and add the same hash.

### Recommendation

We recommend to add this check.

### Status

Double

## Owner can propose themselves as the new owner

### Description

There is no check if the proposed new owner is already owner `_newOwnerProposal == _owner` at the line [contracts/Ownable.sol#L56](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/mysofinance/v2.git/contracts/Ownable.sol#L56).

### Recommendation

We recommend to add this check.

### Status

Double

## Loan repayment transaction may become stale leading to loss of value

### Description

Compared to borrow functions, in the function `repay()` [contracts/peer-to-peer/BorrowerGateway.sol#L150](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/BorrowerGateway.sol#L150), there is no deadline for stale transactions. If the transaction becomes stale and the value of the collateral token falls, it will not be in borrower's interest to repay the loan, but the transaction can still be executed. The borrower may lose value because of it.

### Recommendation

We recommend adding a deadline for loan repays.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/3f44ff54ce34cb156b10a19a988f8964667dfb4c>

## Checks-Effects-Interaction pattern is not followed

### Description

There are several places where the Checks-Effects-Interaction pattern is not followed, i.e. an external call before the state change:

- [contracts/peer-to-peer/BorrowerGateway.sol#L171](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/BorrowerGateway.sol#L171)
- [contracts/peer-to-peer/LenderVaultFactory.sol#L40](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/LenderVaultFactory.sol#L40)
- [contracts/peer-to-peer/wrappers/ERC20/WrappedERC20Impl.sol#L59](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/wrappers/ERC20/WrappedERC20Impl.sol#L59)
- [contracts/peer-to-peer/wrappers/ERC721/WrappedERC721Impl.sol#L51](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/wrappers/ERC721/WrappedERC721Impl.sol#L51)

### Recommendation

We recommend making external calls after state changes to avoid potential reentrancy issues.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/91e95cd94d11c796ee3b52512964be3c321b20d5>

## Vault owner can make himself a signer

### Description

In the function `addSigners()` [contracts/peer-to-peer/LenderVaultImpl.sol#L249](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/LenderVaultImpl.sol#L249), the owner can make themselves a signer. However, in the function `_newOwnerProposalCheck()` [contracts/peer-to-peer/LenderVaultImpl.sol#L439](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/LenderVaultImpl.sol#L439), a signer cannot be made an owner. This is an inconsistency, either the owner should not be able to make themselves a signer or a signer should be able to be the owner.

### Recommendation

We recommend disallowing the owner to make themselves a signer.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/3f44ff54ce34cb156b10a19a988f8964667dfb4c>

## Vault owner can set `minNumOfSigners` higher than the number of signers

### Description

In the function `setMinNumOfSigners()` [contracts/peer-to-peer/LenderVaultImpl.sol#L240](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/LenderVaultImpl.sol#L240), the owner can set the `minNumOfSigners` higher than the number of signers.

### Recommendation

We recommend adding an ability to provide this feature.

### Status

Double

## No checks if loan or repay amount is `0`

### Description

If `minLoan == 0` and due to rounding, the `loanAmount` can be `0` [contracts/peer-to-peer/LenderVaultImpl.sol#L418](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/LenderVaultImpl.sol#L418):

```solidity
loanAmount =
    (loanPerCollUnit * (collSendAmount - expectedTransferFee)) /
    (10 ** IERC20Metadata(generalQuoteInfo.collToken).decimals());
```

If the `collSendAmount != 0` and `loanAmount == 0`, then the borrower will lose collateral tokens for nothing in return.

If the `loanAmount` and/or `interestRateFactor` are small, then `repayAmount` can be `0` [contracts/peer-to-peer/LenderVaultImpl.sol#L436](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/LenderVaultImpl.sol#L436):

```solidity
repayAmount = (loanAmount * interestRateFactor) / Constants.BASE;
```

If the `repayAmount == 0`, the borrow will be a straight swap. It is unclear if this should be allowed.

### Recommendation

We recommend adding checks for `0` where appropriate.

### Status

Double

## Loan proposal can be executed even after the first due date

### Description

After `finalizeLoanTermsAndTransferColl()` is called by the borrower, the loan status is set to `READY_TO_EXECUTE`. In the function `checkAndUpdateStatus()` [contracts/peer-to-pool/LoanProposalImpl.sol#L292](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/LoanProposalImpl.sol#L292), there is no time limit for how long the loan can be in this status. The `executeLoanProposal()` can be called even after the first due timestamp. If `executeLoanProposal()` is called late, the loan can be defaulted immediately after the borrower receives the loan token.

On the other hand, there is `LOAN_EXECUTION_GRACE_PERIOD`

### Recommendation

We recommend ensuring `executeLoanProposal()` is called in time before the first due timestamp.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/e5f6561e042e5615dead4137d8a6f8307fc976a1>

## Factory address can be replaced with `msg.sender`

### Description

Since the factory calls `initialize()` when deploying a funding pool or a loan proposal, the factory address can be replaced with `msg.sender` to save gas in `initialize()` functions: [contracts/peer-to-pool/FundingPoolImpl.sol#L40](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/FundingPoolImpl.sol#L40), [contracts/peer-to-pool/LoanProposalImpl.sol#L70](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/LoanProposalImpl.sol#L70).

### Recommendation

We recommend replacing `_factory` with `msg.sender`.

### Status

Double

## Gas optimizations

### Description

1. In the function `executeLoanProposal()` [contracts/peer-to-pool/FundingPoolImpl.sol#L175](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/FundingPoolImpl.sol#L175), it is possible to save gas by caching storage variables `factory` and `depositToken` into memory variables.

2. In the function `acceptLoanTerms()` [contracts/peer-to-pool/LoanProposalImpl.sol#L121](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/LoanProposalImpl.sol#L121), it is possible to cache `lastLoanTermsUpdateTime`.

3. In the function `unlockCollateral()`, the variable `totalUnlockableColl` can be `0` if the collateral is unlocked from a compartment. In that case there will be an unnecessary write to storage at the line [contracts/peer-to-peer/LenderVaultImpl.sol#L89](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/LenderVaultImpl.sol#L89). It is recommended to check if `totalUnlockableColl > 0` before writing to storage.

4. Caching immutable variable to memory doesn't save gas since reading immutable variable costs the same as reading from memory at the lines [contracts/peer-to-peer/QuoteHandler.sol#L31](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/QuoteHandler.sol#L31), [contracts/peer-to-peer/QuoteHandler.sol#L64](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/QuoteHandler.sol#L64), [contracts/peer-to-peer/QuoteHandler.sol#L298](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/QuoteHandler.sol#L298).

5. In the function `_checkTokensAndCompartmentWhitelist()` [contracts/peer-to-peer/QuoteHandler.sol#L393](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/QuoteHandler.sol#L393), the parameter `_addressRegistry` is unnecessary since `addressRegistry` is already available as an immutable variable.

6. At the line [contracts/peer-to-peer/QuoteHandler.sol#L401](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/QuoteHandler.sol#L401), `collTokenWhitelistState` is used only if there is no compartment so it can be moved inside the if statement.

### Recommendation

We recommend implementing gas optimizations.

### Status

Fixed at  <https://github.com/mysofinance/v2/commit/eac003961e8301505749dc6915f1a8dd42e97cb2>

## No address zero check

### Description

In the function `proposeLoanTerms()` [contracts/peer-to-pool/LoanProposalImpl.sol#L81](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/LoanProposalImpl.sol#L81), there is no check if borrower is `address(0)`.

### Recommendation

We recommend adding an address zero check.

### Status

Double

## No checks if the transfer amount is `0`

### Description

1. In the function `repay()` [contracts/peer-to-pool/LoanProposalImpl.sol#L357](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/LoanProposalImpl.sol#L357), if all collateral token for the current repayment index is converted then `collTokenLeftUnconverted == 0`. In that case, there is no need to transfer loan token at the line [contracts/peer-to-pool/LoanProposalImpl.sol#L393](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/LoanProposalImpl.sol#L393). Also, transfer amount of collateral token can be `0` at the line [contracts/peer-to-pool/LoanProposalImpl.sol#L411](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-pool/LoanProposalImpl.sol#L411).

2. At the line [contracts/peer-to-peer/compartments/staking/CurveLPStakingCompartment.sol#L210](https://github.com/mysofinance/v2/blob/c0536c1ad650805bdf5d68390de0434eb570e694/contracts/peer-to-peer/compartments/staking/CurveLPStakingCompartment.sol#L210), `tokenAmount` can be `0` if collateral was not staked or due to rounding.

### Recommendation

We recommend checking if the transfer amounts are 0 before doing the transfer in this function.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/e5f6561e042e5615dead4137d8a6f8307fc976a1>

## `...gas limit` - High finding can be more optimal for gas

### Description

There's an opportunity for gas optimization in the problem-solving, cause adding a public variable isn't optimal for gas.

### Recommendation

We recommend replacing `uint256 public numRegisteredVaults` with this function:

```solidity
function numRegisteredVaults() external view returns (uint256) {
    return _registeredVaults.length;
}
```

### Status

Fixed at <https://github.com/mysofinance/v2/commit/d9bc593b6cfffd2ce192b459f13dc74918da4d0c>

## Division before multiplication

### Description

At the line [contracts/peer-to-peer/LenderVaultImpl.sol#L465-L471](https://github.com/mysofinance/v2/blob/453dc0c78def6b00370b408c854d8eb7e67206f9/contracts/peer-to-peer/LenderVaultImpl.sol#L465-L471), there is division before multiplication which could lead to loss of precision. `getPrice()` divides collateral token price by loan token price then multiplies by `loanPerCollUnitOrLtv`:

```solidity
loanPerCollUnit =
    (quoteTuple.loanPerCollUnitOrLtv *
        ((priceOfCollToken * 10 ** loanTokenDecimals) /
            priceOfLoanToken)) /
    Constants.BASE;
```

### Recommendation

We recommend returning the raw prices of both tokens from the oracle, then computing the `loanPerCollUnit`.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/3bb725b260b05d27d9733e48aba3fc8642f29bc1>

## Unused `quotePolicyManager` variable in `AddressRegistry`

### Description

The variable `quotePolicyManager` declared at the line [AddressRegistry.sol#L30](https://github.com/mysofinance/v2/blob/141d51423bb831006537fe2ff67f16c6c56e5281/contracts/peer-to-peer/AddressRegistry.sol#L30) is unused in the contract.

### Recommendation

We recommend to remove the unused variable.

### Status

DOUBLE

## The function `copyPublishedOnChainQuote()` does not update `onChainQuoteHistory`

### Description

The function [`copyPublishedOnChainQuote()`](https://github.com/mysofinance/v2/blob/141d51423bb831006537fe2ff67f16c6c56e5281/contracts/peer-to-peer/QuoteHandler.sol#L111) is used to copy published on chain quotes and updates `isOnChainQuoteFromVault`, but it doesn't update `onChainQuoteHistory`. This function can be used by approved senders to bypass writing to `onChainQuoteHistory`.

### Recommendation

We recommend to save `onChainQuote.generalQuoteInfo.validUntil` in `isPublishedOnChainQuote` and update `onChainQuoteHistory` when copying a published quote.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/ac3f28fca0637a46d57a73331ff743dbe61cd366>

## Unnecessary check if quote is swap

### Description

If tenor is zero and quote is a swap, then `quoteTuple.upfrontFeePctInBase == Constants.BASE` based on [QuoteHandler.sol#L591](https://github.com/mysofinance/v2/blob/141d51423bb831006537fe2ff67f16c6c56e5281/contracts/peer-to-peer/QuoteHandler.sol#L591). Then the check at the line [BasicQuotePolicyManager.sol#L299](https://github.com/mysofinance/v2/blob/141d51423bb831006537fe2ff67f16c6c56e5281/contracts/peer-to-peer/policyManagers/BasicQuotePolicyManager.sol#L299) is unnecessary.

### Recommendation

We recommend to move the `if` statement at line [BasicQuotePolicyManager.sol#L299](https://github.com/mysofinance/v2/blob/141d51423bb831006537fe2ff67f16c6c56e5281/contracts/peer-to-peer/policyManagers/BasicQuotePolicyManager.sol#L299) inside the `if` at line [BasicQuotePolicyManager.sol#L283](https://github.com/mysofinance/v2/blob/141d51423bb831006537fe2ff67f16c6c56e5281/contracts/peer-to-peer/policyManagers/BasicQuotePolicyManager.sol#L283).

### Status

Fixed at <https://github.com/mysofinance/v2/commit/ac3f28fca0637a46d57a73331ff743dbe61cd366>

## Inadequate checks for `quoteBounds`

### Description

1. The `quoteBounds` parameters `minLoanPerCollUnitOrLtv` and `maxLoanPerCollUnitOrLtv` can be set to `0`. `quoteTuple` with `loanPerCollUnitOrLtv == 0` is not allowed at the line [QuoteHandler.sol#L560](https://github.com/mysofinance/v2/blob/141d51423bb831006537fe2ff67f16c6c56e5281/contracts/peer-to-peer/QuoteHandler.sol#L560).
2. The `quoteBounds.minFee` checks the min value that `quoteTuple.upfrontFeePctInBase` can be, but `quoteBounds.minFee` can be set to a bigger value than `Constants.BASE` which is the max value for `quoteTuple.upfrontFeePctInBase` ([QuoteHandler.sol#L541-L558](https://github.com/mysofinance/v2/blob/141d51423bb831006537fe2ff67f16c6c56e5281/contracts/peer-to-peer/QuoteHandler.sol#L541-L558)).

### Recommendation

We recommend to add more checks for `quoteBounds`.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/ac3f28fca0637a46d57a73331ff743dbe61cd366>

## Comparison of different units

### Description

The comparison at the lines [BasicQuotePolicyManager.sol#L274-L277](https://github.com/mysofinance/v2/blob/141d51423bb831006537fe2ff67f16c6c56e5281/contracts/peer-to-peer/policyManagers/BasicQuotePolicyManager.sol#L274-L277) can have different units in some scenarios:

1. If the `quoteBounds.minLoanPerCollUnitOrLtv` and `quoteBounds.maxLoanPerCollUnitOrLtv` have `LTV` value in terms of `Constants.BASE`, but an oracle is not required then `quoteTuple.loanPerCollUnitOrLtv` will be in terms of `loanPerCollUnit`.
2. If the `quoteBounds.minLoanPerCollUnitOrLtv` and `quoteBounds.maxLoanPerCollUnitOrLtv` are in terms of `loanPerCollUnit`, but the quote tuple has an oracle then `quoteTuple.loanPerCollUnitOrLtv` will be in terms of `LTV`.

### Recommendation

We recommend to save separate max/min values of `loanPerCollUnit` and `LTV` in `quoteBounds` and compare with `quoteTuple` accordingly.

### Status

Fixed at <https://github.com/mysofinance/v2/commit/ac3f28fca0637a46d57a73331ff743dbe61cd366>
