# Lido Core — Security Context · Scope 2: V3 stVaults + proofs

Load = this file + **the full Scope-load doc set** listed below, read together into one agent context.
**Scope 2 — the V3 stVaults system + the two consensus-layer proof surfaces it shares a trust boundary with.**
The routing table below is the in-scope set (23 contracts); it is the per-contract navigation aid, not the gated
unit (the **Scope load** is). Scope-1 Core-Pool contracts are trusted boundary edges only — see [`AGENTS-1.md`](./AGENTS-1.md).

## Scope load
`00, 06, 07, R, 08, vault-invariants` — the full V3-stVaults set

## Using the docs
- [`00-architecture-overview.md`](./00-architecture-overview.md) is the orientation — module map + the critical flows.
- The routing table maps each in-scope contract to its **primary** module (its home) and **secondary** modules (the seams). `prereqs` = read-first. The whole scope loads in full (see **Scope load** above); per-contract routing remains for targeted navigation within it. A `VaultHub` audit needs only `06`: the Accounting-side bad-debt seam is folded into `06`'s bad-debt flow, so full `03` is not loaded in this scope.
- Claims are cited by **symbol name**, not line numbers, so they resolve against live source.
- **Source-of-truth & precedence:** where docs disagree, the **per-contract module is canonical**. The cross-contract role index lives in [`07` role matrix](./07-governance-permissions.md#role-matrix-high-impact-roles-only) — a derived navigation aid; on any conflict the cited per-contract module wins.
- [`vault-invariants.md`](./vault-invariants.md) is a curated, **supplementary** set of load-bearing properties for the V3-stVaults contracts — part of the Scope load. Treat it as orientation, **not** the audit-target set; keep deriving invariants from source.
- The V3 vault docs live outside `docs/docs/` — in the V3 Technical Paper (`docs/Lido_V3_Whitepaper.pdf`) and the `docs/run-on-lido/stvaults/` tree.

## Routing (contract → modules; `prereqs` = read-first, `—` = none, `00` optional orientation)
| Contract | path | primary | secondary | prereqs | scope |
|---|---|---|---|---|---|
| `StakingVault` | `contracts/0.8.25/vaults/StakingVault.sol` | 06 | R | — | 2 |
| `VaultHub` | `contracts/0.8.25/vaults/VaultHub.sol` | 06 | 03 | — | 2 |
| `VaultFactory` | `contracts/0.8.25/vaults/VaultFactory.sol` | 06 | — | — | 2 |
| `PinnedBeaconProxy` | `contracts/0.8.25/vaults/PinnedBeaconProxy.sol` | 06 | — | — | 2 |
| `PinnedBeaconUtils` | `contracts/0.8.25/vaults/lib/PinnedBeaconUtils.sol` | 06 | — | — | 2 |
| `RefSlotCache` | `contracts/0.8.25/vaults/lib/RefSlotCache.sol` | 06 | — | — | 2 |
| `TriggerableWithdrawals` | `contracts/common/lib/TriggerableWithdrawals.sol` | 06 | R | — | 2 |
| `OperatorGrid` | `contracts/0.8.25/vaults/OperatorGrid.sol` | 06 | 07 | — | 2 |
| `LazyOracle` | `contracts/0.8.25/vaults/LazyOracle.sol` | 06 | 03 | — | 2 |
| `Confirmable2Addresses` | `contracts/0.8.25/utils/Confirmable2Addresses.sol` | 06 | — | — | 2 |
| `Dashboard` | `contracts/0.8.25/vaults/dashboard/Dashboard.sol` | 06 | 07 | — | 2 |
| `Permissions` | `contracts/0.8.25/vaults/dashboard/Permissions.sol` | 06 | 07 | — | 2 |
| `NodeOperatorFee` | `contracts/0.8.25/vaults/dashboard/NodeOperatorFee.sol` | 06 | 07 | — | 2 |
| `AccessControlConfirmable` | `contracts/0.8.25/utils/AccessControlConfirmable.sol` | 06 | — | — | 2 |
| `Confirmations` | `contracts/0.8.25/utils/Confirmations.sol` | 06 | — | — | 2 |
| `PredepositGuarantee` | `contracts/0.8.25/vaults/predeposit_guarantee/PredepositGuarantee.sol` | 06 | R | — | 2 |
| `CLProofVerifier` | `contracts/0.8.25/vaults/predeposit_guarantee/CLProofVerifier.sol` | 06 | R | — | 2 |
| `MeIfNobodyElse` | `contracts/0.8.25/vaults/predeposit_guarantee/MeIfNobodyElse.sol` | 06 | — | — | 2 |
| `ValidatorsExitBus` | `contracts/0.8.9/oracle/ValidatorsExitBus.sol` | 08 | 04 | — | 2 |
| `ValidatorsExitBusOracle` | `contracts/0.8.9/oracle/ValidatorsExitBusOracle.sol` | 08 | 03 | — | 2 |
| `ValidatorExitDelayVerifier` | `contracts/0.8.25/ValidatorExitDelayVerifier.sol` | 08 | R | — | 2 |
| `SSZ` | `contracts/common/lib/SSZ.sol` | R | 06 | — | 2 |
| `GIndex` | `contracts/common/lib/GIndex.sol` | R | 06 | — | 2 |

## Boundary seams
The boundary contract's **internals** are out of scope (trusted / separately audited), but the **seam is in scope**: a finding is valid if it shows the in-scope side mishandling realistic boundary behavior. Stress-test each edge from the in-scope side; assume the other side can be late, wrong, or hostile within reason.

| Seam | in-scope side | Other side |
|---|---|---|
| External-shares mint/burn | `VaultHub.mintShares` / `burnShares` → `Lido.mintExternalShares` / `burnExternalShares` | `Lido` (Scope 1 → [`01`](./01-core-staking.md)) |
| Oracle report → vaults | `VaultHub` gates `LIDO_LOCATOR.accounting()`; `Accounting` reads `badDebtToInternalize()` / calls `decreaseInternalizedBadDebt(...)` | `Accounting` (Scope 1 → [`03`](./03-oracle-accounting.md)) |
| Exit penalty | `ValidatorExitDelayVerifier → stakingRouter.reportValidatorExitDelay` | `StakingRouter` (Scope 1 → [`02`](./02-staking-router-modules.md)) |
| Exit execution | `ValidatorsExitBus → TriggerableWithdrawalsGateway.triggerFullWithdrawals` | `TriggerableWithdrawalsGateway` (Scope 1 → [`04`](./04-withdrawals.md)) |
| Consensus framing | `VaultHub` ctor `IHashConsensus`; `ValidatorsExitBusOracle is BaseOracle` | `HashConsensus`, `BaseOracle` (ingest in [`03`](./03-oracle-accounting.md)) |

Cross-boundary findings that span the in-scope code and these edges are still valid.
