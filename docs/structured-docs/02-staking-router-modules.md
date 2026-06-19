---
doc: "02"
title: Staking Router & Modules
contracts: [NodeOperatorsRegistry, StakingRouter, MinFirstAllocationStrategy]
prereqs: ["01"]
see_also: ["00","03","04","05","07"]
ssot_for: [module-allocation, fee-distribution, validator-counts, exit-reporting]
---
# 02 — Staking Router and Modules

> The core-pool traffic cop. Buffered ETH arrives from [`01` Lido](./01-core-staking.md#core-flows) (after the DSM deposit gate, doc 05); `StakingRouter` allocates it across `IStakingModule` modules, fans oracle-driven fees back, and ingests exited/exit-delay/forced-exit reports. Reports originate in [`03` accounting/oracle](./03-oracle-accounting.md#core-flows); exit verification lives in [`08`](./08-exits.md#core-flows). CSM is a cross-repo module driven only through the `IStakingModule` seam.

## Contracts

| Contract | File | Role |
|---|---|---|
| `StakingRouter` | `0.8.9/StakingRouter.sol` | Module registry; min-first allocation; deposit dispatcher; fee + exited/exit-delay/exit-triggered reporting hub. Behind `OssifiableProxy`. Inherits `BeaconChainDepositor`. |
| `NodeOperatorsRegistry` | `0.4.24/nos/NodeOperatorsRegistry.sol` | Curated module (id=1); redeployed as Simple DVT (id=2). Aragon app (ACL `auth`/`canPerform`, not OZ). Operator/key lifecycle + reward distribution. |
| `MinFirstAllocationStrategy` | `common/lib/MinFirstAllocationStrategy.sol` | Pure allocation library (`allocate` / `allocateToBestCandidate`); fills least-full buckets first. Called from `StakingRouter._getDepositsAllocation`. |
| boundary `IStakingModule` | `common/interfaces/IStakingModule.sol` | The seam every module implements; CSM (cross-repo) plugs in here — see CSM seam below. |
| boundary `BeaconChainDepositor` | `0.8.9/BeaconChainDepositor.sol` | Mixin: `IDepositContract.deposit` in 32-ETH batches. |

## Core flows

### 1. Stake allocation — the min-first capacity formula (load-bearing)

`Lido.deposit` (only DSM may reach it) calls `StakingRouter.deposit{value: N×32 ETH}(depositsCount, moduleId, calldata)`. The count was already clamped by Lido reading `getStakingModuleMaxDepositsCount`, which runs the same allocation math.

```text
StakingRouter.deposit(depositsCount, moduleId, calldata)  (payable, msg.sender == Lido)
  → require getWithdrawalCredentials() != 0               // else EmptyWithdrawalsCredentials
  → require module.status == Active                        // else StakingModuleNotActive
  → require msg.value == depositsCount * DEPOSIT_SIZE       // else InvalidDepositsValue
  → _updateModuleLastDepositState(module, value)           // STATE FIRST (CEI; reentrancy guard)
  → module.obtainDepositData(depositsCount, calldata)      // EXT: pops vetted keys → (pubkeys, sigs)
  → _makeBeaconChainDeposits32ETH(...)                     // EXT: DEPOSIT_CONTRACT.deposit per validator
  → assert balanceBefore - balanceAfter == value           // ALL ETH must leave; partial placement is impossible
External: IStakingModule, beacon DepositContract.
```

Allocation rationale (`_getDepositsAllocation` → `MinFirstAllocationStrategy.allocate`): the router treats each module as a bucket. For module i, with the post-deposit estimate `totalActive' = Σ activeValidators + depositsToAllocate`:

```text
capacity[i] = min( stakeShareLimit[i] * totalActive' / TOTAL_BASIS_POINTS ,   // share-of-protocol cap
                   activeValidators[i] + availableValidators[i] )             // physical key supply
bucket[i]   = activeValidators[i]                                              // current fill
```

`allocate` greedily tops up the *least-filled* bucket with free space until `depositsToAllocate` is exhausted (equalization step in Internal mechanics). Effect: a module below its share target fills first; one over its share is bypassed. `getStakingModuleMaxDepositsCount(id, ether)` returns `newAllocation[i] − activeValidators[i]` — the marginal deposits this module can absorb. Gotcha: `availableValidators` is `depositableValidatorsCount` from `getStakingModuleSummary` (forced to 0 unless `Active`); it caps allocation regardless of share head-room, so empty depositable inventory silently yields zero even under target.

### 2. Module registration and config

`addStakingModule(name, addr, stakeShareLimit, priorityExitShareThreshold, moduleFee, treasuryFee, maxDepositsPerBlock, minDepositBlockDistance)` (`STAKING_MODULE_MANAGE_ROLE`). `addStakingModule` validates address non-zero, name non-empty and `≤ MAX_STAKING_MODULE_NAME_LENGTH`, count `< MAX_STAKING_MODULES_COUNT`, and address not already registered. It then delegates the param bounds to `_updateStakingModule`: `stakeShareLimit ≤ TOTAL_BASIS_POINTS`; `priorityExitShareThreshold ≤ TOTAL_BASIS_POINTS` and `≥ stakeShareLimit`; `moduleFee + treasuryFee ≤ TOTAL_BASIS_POINTS`; `0 < minDepositBlockDistance ≤ uint64.max`; `maxDepositsPerBlock ≤ uint64.max`. New module starts `Active` but `_updateModuleLastDepositState(…, 0)` simulates a zero deposit so DSM cannot deposit into it in the same block it was added. `updateStakingModule` mutates the same fields under the same role. `setStakingModuleStatus` flips `Active`/`DepositsPaused`/`Stopped`.

### 3. Fee distribution — reportRewardsMinted fan-out (load-bearing)

During an oracle report, `Accounting` reads `getStakingRewardsDistribution()` (view) to size the mint, mints the total fee stETH shares to *itself* (`Lido.mintShares(address(this), …)`) then `transferShares` to each module address and the treasury, and finally calls `reportRewardsMinted(moduleIds, totalShares)` (`REPORT_REWARDS_MINTED_ROLE`) so each module arms its internal split.

```text
getStakingRewardsDistribution()  (per module with activeValidators > 0, regardless of status; a Stopped module's slice routes to treasury)
  share_i      = activeValidators_i * FEE_PRECISION_POINTS / totalActiveValidators
  moduleFee_i  = share_i * stakingModuleFee_i / TOTAL_BASIS_POINTS    // zeroed if module Stopped
  treasuryFee  += share_i * treasuryFee_i / TOTAL_BASIS_POINTS + moduleFee_i (into totalFee)
  assert totalFee <= FEE_PRECISION_POINTS                              // protocol fee <= 100%

Accounting → Lido.mintShares(address(this), totalFeeShares) → transferShares(moduleAddr_i / treasury)  // mint to Accounting, then distribute
Accounting → StakingRouter.reportRewardsMinted(ids, totalShares)
  → per module: module.onRewardsMinted(totalShares)                   // EXT; try/catch (see mechanics)
External: each IStakingModule.
```

Subtlety: for a `Stopped` module the per-module fee is *not paid to the module* (`stakingModuleFees[i]` stays 0) but is still added into `totalFee`, so that slice routes to treasury — the DAO can later compensate. `onRewardsMinted` in NOR only flips reward state to `TransferredToModule`; it does NOT distribute. Payout is the permissionless `distributeReward()` (flow 5).

### 4. Exited / exit-delay / forced-exit reporting (load-bearing)

Four distinct callers feed `StakingRouter`; each is a separate seam to stress-test. The exited-count update is a two-phase oracle process.

```text
AccountingOracle.submitReportData
  → updateExitedValidatorsCountByStakingModule(ids, counts)   (REPORT_EXITED_VALIDATORS_ROLE)
      // phase 1: module aggregates. counts must NOT decrease (ExitedValidatorsCountCannotDecrease)
      // and counts <= totalDeposited (ReportedExitedValidatorsExceedDeposited). One call per frame.
AccountingOracle.submitReportExtraDataList
  → reportStakingModuleExitedValidatorsCountByNodeOperator(id, noIds, counts)  (same role)
      // phase 2: per-operator, itemType=2 (EXTRA_DATA_TYPE_EXITED_VALIDATORS), repeatable per module
      → module.updateExitedValidatorsCount(noIds, counts)    // EXT
  → onValidatorsCountsByNodeOperatorReportingFinished()       (same role)
      // closes phase 2: per module, IF module-summary exited == router's stored count,
      → module.onExitedAndStuckValidatorsCountsUpdated()      // EXT; arms reward distribution
ValidatorExitDelayVerifier
  → reportValidatorExitDelay(id, noId, proofSlotTs, pubkey, eligibleToExitInSec)  (REPORT_VALIDATOR_EXITING_STATUS_ROLE)
      → module.reportValidatorExitDelay(...)                  // EXT; records exit-delay report (NOR: event-only, no reward penalty)
TriggerableWithdrawalsGateway
  → onValidatorExitTriggered(exitData[], paidFee, exitType)   (REPORT_VALIDATOR_EXIT_TRIGGERED_ROLE)
      → module.onValidatorExitTriggered(...)                  // EXT; per-entry try/catch
External: AccountingOracle, ValidatorExitDelayVerifier, TriggerableWithdrawalsGateway, each IStakingModule.
```

`exitType` classifies why the exit was triggered and is forwarded unchanged to `module.onValidatorExitTriggered`; each module interprets it per its own implementation (module-side handling out of scope — CSM seam).

Phase ordering matters: phase-1 router totals drive *allocation and fee weight* immediately, but the module learns per-operator counts only in phase 2, then reconciles in `onExitedAndStuckValidatorsCountsUpdated`. If phase 2 spills into the next frame, the router emits `StakingModuleExitedValidatorsIncompleteReporting` and the module carries stale per-operator data for a frame — each module must tolerate this. `unsafeSetExitedValidatorsCount(...)` (`UNSAFE_SET_EXITED_VALIDATORS_ROLE`) is the DAO escape hatch: bypasses the monotonic/non-decrease invariant against expected current values, flagged unsafe in source.

**Stuck validators (deprecated):** there is **no** `reportStakingModuleStuckValidatorsCountByNodeOperator` and no stuck path through the router. `AccountingOracle` extra-data `itemType=1` (legacy `EXTRA_DATA_TYPE_STUCK_VALIDATORS`) now reverts `DeprecatedExtraDataType`. In NOR, `getNodeOperatorSummary` hardcodes `stuckValidatorsCount = 0` (and `refundedValidatorsCount = 0`, `stuckPenaltyEndTimestamp = 0`); `getStuckPenaltyDelay()` returns 0. The exit-delay penalty model replaces the old stuck/refunded counters.

### 5. NOR operator and key lifecycle, reward distribution

NOR has three principals: the DAO (Aragon roles), the node operator (self-manages its own keys), and `StakingRouter` (`STAKING_ROUTER_ROLE`).

```text
DAO admin (MANAGE_NODE_OPERATOR_ROLE):
  addNodeOperator / activate / deactivate / setName / setRewardAddress
  invalidateReadyToDepositKeysRange(from,to)
  setExitDeadlineThreshold(threshold, lateReportingWindow)        // exit-delay penalty params
vetting (SET_NODE_OPERATOR_LIMIT_ROLE, authP by id):
  setNodeOperatorStakingLimit(id, vettedKeysCount)                // the DAO vetting lever
operator OR MANAGE_SIGNING_KEYS:
  addSigningKeys / addSigningKeysOperatorBH / removeSigningKey(s)
  → each calls _increaseValidatorsKeysNonce()                     // invalidates stale DSM signatures
driven by StakingRouter (STAKING_ROUTER_ROLE):
  obtainDepositData → updateExitedValidatorsCount → unsafeUpdateValidatorsCount
  onExitedAndStuckValidatorsCountsUpdated → onRewardsMinted → onWithdrawalCredentialsChanged
  onValidatorExitTriggered → reportValidatorExitDelay → decreaseVettedSigningKeysCount
permissionless:
  distributeReward()  require state == ReadyForDistribution
    → _distributeRewards()  → StETH.transferShares to each active operator rewardAddress
```

Reward state machine: `onRewardsMinted` → `TransferredToModule`; `onExitedAndStuckValidatorsCountsUpdated` → `ReadyForDistribution`; `distributeReward` (anyone) → `Distributed`. `_distributeRewards` skips shares `< 2` (avoids zero-transfer on penalty rounding). `onWithdrawalCredentialsChanged` invalidates all undeposited keys protocol-wide.

## Internal mechanics

- **MinFirstAllocationStrategy equalization.** `allocateToBestCandidate` finds the least-filled bucket with free space, counts ties, caps the top-up at `min( ties>1 ? ceilDiv(size, ties) : size , min(nextLargerBucketValue, capacity) − bucketValue )`. The "next larger bucket value" bound never overshoots a bucket past the next level up, keeping fills balanced across equally-under-target modules. `allocate` loops until `allocated == size` or a pass allocates 0 (all at capacity). Pure library; mutates `buckets` in place. `MAX_UINT256 = 2**256 − 1` = "no candidate" sentinel.
- **CEI / reentrancy in `deposit`.** State (`lastDepositAt`/`lastDepositBlock`) is written *before* the external `obtainDepositData`/deposit calls even though modules are trusted; the `balanceBefore − balanceAfter == value` assert guarantees no ETH is stranded and a module cannot under-consume.
- **try/catch on module fan-out.** `reportRewardsMinted`, `onValidatorsCountsByNodeOperatorReportingFinished`, `onValidatorExitTriggered` wrap the module call in `try/catch`: a non-empty revert is swallowed and surfaced as an event (`RewardsMintedReportFailed`/`ExitedAndStuckValidatorsCountsUpdateFailed`/`StakingModuleExitNotificationFailed`) so one bad module cannot brick the report; an **empty** revert is treated as out-of-gas and re-thrown `UnrecoverableModuleError` (prevents gas-estimation binary-search returning a bogus value).
- **StakingModule struct (packed).** `id (uint24)`, `stakingModuleFee/treasuryFee/stakeShareLimit/priorityExitShareThreshold (uint16)`, `status (uint8)`, `maxDepositsPerBlock/minDepositBlockDistance/lastDepositAt (uint64)`, plus `stakingModuleAddress`, `name`, `lastDepositBlock`, `exitedValidatorsCount`. The router's `exitedValidatorsCount` is the phase-1 aggregate and can legitimately differ from the module summary mid-frame.
- **Shared (moduleId, nodeOperatorId) namespace.** One id space spans NOR (id=1), Simple DVT (id=2), CSM (id=3+). Core code that hardcodes `id == 1` or assumes NOR semantics breaks for other modules.
- **NOR is Aragon ACL, not OZ.** Auth via `_auth(role)`/`canPerform`/`authP` (by operator id), distinct from `StakingRouter`'s OZ `AccessControlEnumerable`. `MAX_NODE_OPERATORS_COUNT = 200` bounds storage iteration. `reportValidatorExitDelay` dedupes by `keccak256(pubkey)` (idempotent), requires `eligibleToExitInSec ≥ exitDeadlineThreshold` and `proofSlotTimestamp - eligibleToExitInSec ≥ exitPenaltyCutoffTimestamp()` (the eligibility-start timestamp, not the proof slot itself, must be at/after the cutoff).

## External interactions

```text
StakingRouter
  ← Lido.deposit                              (only ETH-bearing entry)
  ← AccountingOracle.submitReportData          (updateExitedValidatorsCountByStakingModule)
  ← AccountingOracle.submitReportExtraDataList (reportStakingModuleExited…ByNodeOperator, finished)
  ← ValidatorExitDelayVerifier                 (reportValidatorExitDelay)
  ← TriggerableWithdrawalsGateway              (onValidatorExitTriggered)
  ← DepositSecurityModule                      (decreaseStakingModuleVettedKeysCountByNodeOperator)
  ← Accounting                                 (reportRewardsMinted; reads getStakingRewardsDistribution)
  ← DAO                                        (addStakingModule / updateStakingModule / status / WC / target limits)
  → IStakingModule.obtainDepositData / onRewardsMinted / onExitedAndStuckValidatorsCountsUpdated
  → IStakingModule.onValidatorExitTriggered / reportValidatorExitDelay / updateExitedValidatorsCount
  → IStakingModule.getStakingModuleSummary / getNodeOperatorSummary   (views)
  → IDepositContract.deposit                   (via BeaconChainDepositor)
NodeOperatorsRegistry
  → StETH.transferShares                       (deliver reward shares to operators)
  → LidoLocator.lido()                         (resolve StETH)
```

### CSM seam (boundary — module out of repo)

CSM is the third major module (`lidofinance/community-staking-module`); only the `IStakingModule` interface is in-repo. `StakingRouter` drives it exclusively through that interface — `obtainDepositData`, `onRewardsMinted`, `onExitedAndStuckValidatorsCountsUpdated`, `onValidatorExitTriggered`, `reportValidatorExitDelay`, `decreaseVettedSigningKeysCount`, plus the summary views. Seam invariants to preserve under any change to the router, the extra-data decoder ([`03`](./03-oracle-accounting.md#core-flows)), VEBO ([`08`](./08-exits.md#external-interactions)) / TWG ([`04`](./04-withdrawals.md#external-interactions)): call signatures, return shapes, event order; idempotent `reportValidatorExitDelay`; the shared id namespace. CSM holds operator bond as stETH **shares**, so a change to share-rate semantics or `Burner` ordering ([`03`](./03-oracle-accounting.md#core-flows)) can mis-value bond. CSM internals (bond curve, strikes/performance oracle, `CSEjector`→TWG forced exits, `CSVerifier` proofs, V2 gates) are out of scope.

> **Invariants:** see [`core-invariants.md` §4](./core-invariants.md#4-staking-router-and-allocations) — supplementary, not the full set; derive others from source.

## Key constants

| Constant | Value | Purpose |
|---|---|---|
| `DEPOSIT_SIZE` | 32 ETH | Beacon-chain deposit unit (via `BeaconChainDepositor`) |
| `TOTAL_BASIS_POINTS` | 10000 | BP denominator for fees, share limits, exit threshold |
| `FEE_PRECISION_POINTS` | 10**20 | High-precision fee base = 100 × 10**18 (= 100%) |
| `MAX_STAKING_MODULES_COUNT` | 32 | Max registered modules |
| `MAX_STAKING_MODULE_NAME_LENGTH` | 31 | Module-name byte cap |
| `MAX_NODE_OPERATORS_COUNT` | 200 | NOR per-module operator cap (bounds storage iteration) |
| `MAX_UINT256` | 2**256 − 1 | `MinFirstAllocationStrategy` no-candidate sentinel |
| `MAX_STUCK_PENALTY_DELAY` | 365 days | Legacy NOR constant; stuck-penalty logic removed |

## Source references

**Live source** (every symbol cited inline above resolves against these files):
- `0.8.9/StakingRouter.sol` — registry, `_getDepositsAllocation`/`getStakingModuleMaxDepositsCount`/`deposit`, `getStakingRewardsDistribution`/`reportRewardsMinted`, exited/exit-delay/exit-triggered reporting, `unsafeSetExitedValidatorsCount`, OZ roles.
- `0.4.24/nos/NodeOperatorsRegistry.sol` — Aragon-ACL Curated/Simple-DVT; operator & key lifecycle, reward state machine + `_distributeRewards`, `getNodeOperatorSummary` (stuck hardcoded 0), `reportValidatorExitDelay`/`setExitDeadlineThreshold`.
- `common/lib/MinFirstAllocationStrategy.sol` — `allocate`/`allocateToBestCandidate`. Boundary: `0.8.9/BeaconChainDepositor.sol`, `common/interfaces/IStakingModule.sol`.

**Official docs (docs/docs/):** `contracts/staking-router.md`, `contracts/node-operators-registry.md`, `staking-modules/csm/` (CSM impl in `lidofinance/community-staking-module`).
