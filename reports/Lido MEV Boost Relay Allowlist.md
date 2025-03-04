# CRITICAL

Not found

# HIGH

Not found

# MEDIUM

Not found

# INFORMATIONAL

## Function `_safe_erc20_transfer()` doesn't revert if `token` is not a contract

### Description

At the line https://github.com/lidofinance/mev-boost-relay-allowed-list/blob/26ec6791c2466e784a894b8867db71d8de620745/contracts/MEVBoostRelayAllowedList.vy#L288.

The function `_safe_erc20_transfer()` takes address `token` as argument, but it doesn't check if that address is a contract. A low-level call to an EOA address returns `True` with no return data. This can lead to an emitting of the event `ERC20Recovered()` even though no tokens were transferred.

### Recommendation

It is recommended to check if `token` address is a contract in the function `_safe_erc20_transfer()`.

### Status
Fixed at https://github.com/lidofinance/mev-boost-relay-allowed-list/commit/293636247bf6fe73153f794cb9b807355019eb09

## Inaccurate NatSpec comments

### Description

At the line https://github.com/lidofinance/mev-boost-relay-allowed-list/blob/26ec6791c2466e784a894b8867db71d8de620745/contracts/MEVBoostRelayAllowedList.vy#L164, it should be "Remove relay" instead of "Add relay".

At the line https://github.com/lidofinance/mev-boost-relay-allowed-list/blob/26ec6791c2466e784a894b8867db71d8de620745/contracts/MEVBoostRelayAllowedList.vy#L189, it should be "Address of the new owner".

At the line https://github.com/lidofinance/mev-boost-relay-allowed-list/blob/26ec6791c2466e784a894b8867db71d8de620745/contracts/MEVBoostRelayAllowedList.vy#L232, the `recipient` address is not necessarily the DAO treasury. The comment should reflect that.

### Recommendation

It is recommended to fix these comments.

### Status
Fixed at https://github.com/lidofinance/mev-boost-relay-allowed-list/commit/293636247bf6fe73153f794cb9b807355019eb09

## Event name `RelaysUpdated` is ambiguous

### Description

At the line https://github.com/lidofinance/mev-boost-relay-allowed-list/blob/26ec6791c2466e784a894b8867db71d8de620745/contracts/MEVBoostRelayAllowedList.vy#L21, the event name `RelaysUpdated` can be interpreted as some relays in the list were updated. It is recommended to name the event `AllowedListUpdated` or `RelayAllowedListUpdated`.

### Recommendation

Consider changing the name of this event to better reflect the state change.

### Status
Fixed at https://github.com/lidofinance/mev-boost-relay-allowed-list/commit/293636247bf6fe73153f794cb9b807355019eb09
