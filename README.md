# Smart Contract Security Audit Portfolio

This repository contains my smart contract security audit reports for DeFi projects from [Statemind public audits](https://github.com/statemindio/public-audits). Each report provides a detailed analysis of the smart contract implementations, identifying potential vulnerabilities, security concerns, and recommendations for improvements.

## Audit Reports

### DeFi Protocols
- [1inch V5](./reports/1inch%20V5.md)
- [Arrakis V2](./reports/Arrakis%20V2%20Core.md)
  - [Core Update](./reports/Arrakis%20V2%20Core%20Update.md)
  - [Manager Templates](./reports/ArrakisV2%20Manager%20Templates.md)
  - [Periphery](./reports/ArrakisV2Periphery.md)
- [Buttonswap Core](./reports/ButtonswapCore.md)
- [Instadapp AvocadoV3](./reports/Instadapp%20AvocadoV3.md)
- [Keep3r Sidechain Oracles](./reports/Keep3rSidechainOracles.md)
- [Myso V2](./reports/MysoV2.md)

### Lido Protocol
- [Lido V2](./reports/LidoV2.md)
- [Easy Track](./reports/LidoEasyTrack.md)
- [Insurance Fund](./reports/LidoInsuranceFund.md)
- [MEV Boost Relay Allowlist](./reports/Lido%20MEV%20Boost%20Relay%20Allowlist.md)
- [TRP Vesting Escrow](./reports/Lido%20TRPVestingEscrow.md)
- [Yearn veYFI](./reports/YearnVeYFI.md)

## Finding Severity Breakdown

Each vulnerability discovered during the audit is classified based on their severity and likelihood and have the following classification:

| Severity | Description |
| --- | --- |
| Critical | Bugs leading to assets theft, funds locking, or any other issues that lead to fund loss. |
| High | Bugs that break contract core logic or lead to the contract state corruption, or any bugs that require manual recovery. |
| Medium | Bugs leading to partial failure of the contract minor logic under specific conditions or significant gas optimizations. |
| Informational | Bugs, suggestions, or optimizations that do not have a significant impact in terms of contract security. |