# Lido Core — Security Context · Scope 1: Core Pool

Load = this file + **the full Scope-load doc set** listed below, read together into one agent context.
**Scope 1 — the Core Pool: staking, withdrawals, and oracle report execution.** The routing table below is the
in-scope set (16 contracts); it is the per-contract navigation aid, not the gated unit (the **Scope load** is).
Scope-2 vault contracts are trusted boundary edges only — see [`AGENTS-2.md`](./AGENTS-2.md).

## Scope load
`00, 01, 02, 03, 04, 05` — the full Core-Pool set

## Using the docs
- [`00-architecture-overview.md`](./00-architecture-overview.md) is the orientation — module map + the critical flows + the report-execution narrative (its SSOT).
- The routing table maps each in-scope contract to its **primary** module (its home) and **secondary** modules (the seams). `prereqs` = read-first. The whole scope loads in full (see **Scope load** above; gated **≤ 30k tokens**); per-contract routing remains for targeted navigation within it.
- Claims are cited by **symbol name**, not line numbers, so they resolve against live source.
- **Source-of-truth & precedence:** where docs disagree, the **per-contract module is canonical**. The cross-contract role index lives in [`07` role matrix](./07-governance-permissions.md#role-matrix-high-impact-roles-only) — it is a derived navigation aid; on any conflict the cited per-contract module wins.

## Routing (contract → modules; `prereqs` = read-first, `—` = none, `00` optional orientation)
| Contract | path | primary | secondary | prereqs | scope |
|---|---|---|---|---|---|
| `Lido` | `contracts/0.4.24/Lido.sol` | 01 | 03 | — | 1 |
| `StETH` | `contracts/0.4.24/StETH.sol` | 01 | — | — | 1 |
| `StakeLimitUtils` | `contracts/0.4.24/lib/StakeLimitUtils.sol` | 01 | — | — | 1 |
| `NodeOperatorsRegistry` | `contracts/0.4.24/nos/NodeOperatorsRegistry.sol` | 02 | 07 | 01 | 1 |
| `StakingRouter` | `contracts/0.8.9/StakingRouter.sol` | 02 | — | 01 | 1 |
| `MinFirstAllocationStrategy` | `contracts/common/lib/MinFirstAllocationStrategy.sol` | 02 | — | 01 | 1 |
| `Accounting` | `contracts/0.8.9/Accounting.sol` | 03 | 04 | 00 | 1 |
| `Burner` | `contracts/0.8.9/Burner.sol` | 03 | 04 | 00 | 1 |
| `OracleReportSanityChecker` | `contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol` | 03 | — | 00 | 1 |
| `WithdrawalQueue` | `contracts/0.8.9/WithdrawalQueue.sol` | 04 | 03 | — | 1 |
| `WithdrawalQueueBase` | `contracts/0.8.9/WithdrawalQueueBase.sol` | 04 | — | — | 1 |
| `WithdrawalQueueERC721` | `contracts/0.8.9/WithdrawalQueueERC721.sol` | 04 | — | — | 1 |
| `WithdrawalVault` | `contracts/0.8.9/WithdrawalVault.sol` | 04 | R | — | 1 |
| `WithdrawalVaultEIP7002` | `contracts/0.8.9/WithdrawalVaultEIP7002.sol` | 04 | R | — | 1 |
| `TriggerableWithdrawalsGateway` | `contracts/0.8.9/TriggerableWithdrawalsGateway.sol` | 04 | R | — | 1 |
| `ExitLimitUtils` | `contracts/0.8.9/lib/ExitLimitUtils.sol` | 04 | — | — | 1 |

## Boundary seams
The boundary contract's **internals** are out of scope (trusted / separately audited), but the **seam is in scope**: a finding is valid if it shows the in-scope side mishandling realistic boundary behavior. Stress-test each edge from the in-scope side; assume the other side can be late, wrong, or hostile within reason.

| Seam | in-scope side | Other side |
|---|---|---|
| External-shares mint/burn | `Lido.mintExternalShares` / `burnExternalShares` | `VaultHub` (Scope 2 → [`06`](./06-vaults.md)) |
| Vault bad-debt internalize | `Accounting` reads `vaultHub.badDebtToInternalize()`, then `vaultHub.decreaseInternalizedBadDebt(...)` + `Lido.internalizeExternalBadDebt(...)` | `VaultHub` (Scope 2 → [`06`](./06-vaults.md)) |
| Report ingestion | `Accounting.handleOracleReport` gated to `accountingOracle` | `AccountingOracle` (ingest folded into [`03`](./03-oracle-accounting.md)) |
| Deposit gate | `DepositSecurityModule` gates `Lido.deposit` | `DepositSecurityModule` ([`05`](./05-deposit-security.md)) |
| Exit penalty in | `StakingRouter.reportValidatorExitDelay` | `ValidatorExitDelayVerifier` (Scope 2 → [`08`](./08-exits.md)) |
| Exit execution in | `ValidatorsExitBus → TriggerableWithdrawalsGateway.triggerFullWithdrawals` | `ValidatorsExitBus` (Scope 2 → [`08`](./08-exits.md)) |

Cross-boundary findings that span the in-scope code and these edges are still valid.
