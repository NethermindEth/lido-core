---
doc: "08"
title: Validator Exits
contracts: [ValidatorsExitBus, ValidatorsExitBusOracle, ValidatorExitDelayVerifier]
prereqs: ["04"]
see_also: ["04","R","02","03"]
ssot_for: [exit-bus-publication, exit-delay-proof, triggerable-exits-veb-side]
---
# 08 — Validator Exits

> The validator-exit machinery: `ValidatorsExitBus` (VEB) — the packed exit-request store + escalation to the gateway; `ValidatorsExitBusOracle` (VEBO) — the consensus-gated oracle that publishes exit requests; and `ValidatorExitDelayVerifier` (VEDV) — the permissionless EIP-4788 + SSZ proof of a stalled exit. VEB escalates forced exits to the `TriggerableWithdrawalsGateway`; the EIP-7002 fee/refund + predeploy mechanics live in [`04`](./04-withdrawals.md#core-flows). SSZ / EIP-4788 proof model in [`R`](./R-consensus-proof-reference.md#distilled-external-specs); the exit-delay penalty is applied in the module ([`02`](./02-staking-router-modules.md#core-flows)); VEBO ingests via the same `BaseOracle`/`HashConsensus` machinery as Accounting ([`03`](./03-oracle-accounting.md#core-flows)).

## Contracts

| Contract | File | Role |
|---|---|---|
| `ValidatorsExitBus` (VEB) | `0.8.9/oracle/ValidatorsExitBus.sol` | Packed 64-byte exit-request store/parser; hash registry; emits `ValidatorExitRequest`; escalates to TWG. |
| `ValidatorsExitBusOracle` (VEBO) | `0.8.9/oracle/ValidatorsExitBusOracle.sol` | `is BaseOracle, ValidatorsExitBus` — consensus-gated ingestion (separate committee, faster frame). |
| `ValidatorExitDelayVerifier` (VEDV) | `0.8.25/ValidatorExitDelayVerifier.sol` | Permissionless EIP-4788+SSZ proof of a stalled exit; reports delay to `StakingRouter`. |
| `TriggerableWithdrawalsGateway` (boundary) | `0.8.9/TriggerableWithdrawalsGateway.sol` | Executes the EIP-7002 full exit VEB escalates to — fee/refund + predeploy in [`04`](./04-withdrawals.md#core-flows). |
| `StakingRouter`/`OracleReportSanityChecker` | (boundary) | exit-delay-penalty + exit-triggered sink ([`02`](./02-staking-router-modules.md#core-flows)); VEBO report damage bound ([`03`](./03-oracle-accounting.md#core-flows)). |

## Core flows

### 1. Triggerable EIP-7002 forced exit — VEB side (LOAD-BEARING)
The VEB resolves packed exit requests from a previously delivered hash (registry in Internal mechanics) and escalates a full exit to the gateway; the EIP-7002 fee/refund, per-pubkey predeploy call, and `StakingRouter.onValidatorExitTriggered` all run inside TWG ([`04`](./04-withdrawals.md#core-flows)).
```text
VEB.triggerExits(exitsData, exitDataIndexes, refundRecipient)  // payable, whenResumed, preservesEthBalance, PERMISSIONLESS
  → require msg.value>0; indexes non-empty + strictly increasing; hash delivered; moduleId!=0 (checked per selected index)
  → TWG.triggerFullWithdrawals{value: msg.value}(validatorsData[], refundRecipient, EXIT_TYPE)   // → 04: gateway fee/refund + EIP-7002 predeploy
External: TriggerableWithdrawalsGateway (→ 04).
```
`triggerExits` is permissionless — its teeth come from the staged-and-**delivered** hash gate (the request payload must match a hash already published through the consensus/staged path, flow 2), not from a role. `exitDataIndexes` must be strictly increasing within range.

### 2. VEBO exit-request publication and hash path
```text
VEBO.submitReportData(data, contractVersion)   // whenResumed; SUBMIT_DATA_ROLE or consensus member
  → _checkConsensusData(refSlot/version/hash)   // BaseOracle seam → 03/00
  → _storeOracleExitRequestHash(hash, ver): delivered timestamp = _getTime()   // VEDV consumes this
  → checkExitBusOracleReport(requestsCount)      // EXT: OracleReportSanityChecker damage bound → 03
  → per request emit ValidatorExitRequest(moduleId, nodeOpId, valIndex, pubkey, timestamp)
Hash-first: SUBMIT_REPORT_HASH_ROLE → submitExitRequestsHash(hash); then anyone → submitExitRequestsData(req)
  → keccak must match a staged, undelivered hash; _consumeLimit (sliding window); emit; mark delivered.
```
VEBO is `AccountingOracle`'s twin (same `BaseOracle`/`HashConsensus` machinery, **separate committee + faster frame** to expedite exits) — consensus/quorum/frame model documented once in [`00`](./00-architecture-overview.md#the-critical-flows). Only the payload differs: `DATA_FORMAT_LIST = 1`, each request exactly 64 bytes `(uint24 moduleId ‖ uint40 nodeOpId ‖ uint64 valIndex ‖ bytes48 pubkey)`, strictly ascending by `(moduleId,nodeOpId,valIndex)` (else `InvalidRequestsDataSortOrder`); `moduleId==0` invalid. The emitted event is **advisory**; on-chain teeth come from flows 1 and 3.

### 3. Permissionless exit-delay proof — VEDV (LOAD-BEARING)
When a validator requested to exit via VEBO has its CL `exitEpoch` still unset past the per-operator threshold, **anyone** can prove the stall and report it for penalty:
```text
caller → ValidatorExitDelayVerifier.verifyValidatorExitDelay(beaconBlock, witnesses[], exitRequests)
  → _verifyBeaconBlockRoot: slot >= FIRST_SUPPORTED_SLOT; BEACON_ROOTS.staticcall(rootsTimestamp)  // EXT: EIP-4788
       require decoded root == header.hashTreeRoot()
  → deliveredTimestamp = veb.getDeliveryTimestamp(keccak(exitRequests))   // VEB
  → proofSlotTimestamp = GENESIS_TIME + slot*SECONDS_PER_SLOT
  → per witness:
      (pubkey,nodeOpId,moduleId,valIndex) = veb.unpackExitRequest(...)
      eligibleToExitInSec = _getSecondsSinceExitIsEligible(deliveredTimestamp, activationEpoch, proofSlotTimestamp)
      _verifyValidatorExitUnset: SSZ.verifyProof of Validator{exitEpoch = FAR_FUTURE_EPOCH, ...} under header.stateRoot
      → StakingRouter.reportValidatorExitDelay(moduleId, nodeOpId, proofSlotTimestamp, pubkey, eligibleToExitInSec)  // EXT → 02
External: EIP-4788 BEACON_ROOTS, StakingRouter.
```
The proof's force is the **`FAR_FUTURE_EPOCH` invariant**: the witness `Validator` leaf is reconstructed with `exitEpoch` *hard-coded* to `FAR_FUTURE_EPOCH = type(uint64).max`, so a passing SSZ proof against the EIP-4788-attested `stateRoot` is exactly "this validator has not scheduled an exit at this slot." Eligibility = later of VEBO delivery time and earliest *voluntary* exit (`GENESIS_TIME + activationEpoch*SLOTS_PER_EPOCH*SECONDS_PER_SLOT + SHARD_COMMITTEE_PERIOD_IN_SECONDS`); `proofSlotTimestamp <= eligible` reverts `ExitIsNotEligibleOnProvableBeaconBlock`. `verifyHistoricalValidatorExitDelay` proves the same against an older block via `historical_summaries` (Capella+, beyond the 8192-slot EIP-4788 buffer). VEDV holds **no roles** — trust is the SSZ proof + EIP-4788 root; threshold + penalty live in the module ([`02`](./02-staking-router-modules.md#core-flows)). SSZ/GIndex + EIP-4788 model in [`R`](./R-consensus-proof-reference.md#distilled-external-specs); how the CL schedules an accepted exit (churned `exit_epoch`) and the `activation_epoch + SHARD_COMMITTEE_PERIOD` eligibility floor this proof reconstructs are in [`R`](./R-consensus-proof-reference.md#consensus-layer-request-processing-the-seam).

### 4. Admin — pause, resume, rate-limit
```text
PAUSE_ROLE  → VEBO.pauseFor/pauseUntil   (GateSeal can pause)
RESUME_ROLE → VEBO.resume                (deployed paused)
EXIT_REQUEST_LIMIT_MANAGER_ROLE → VEB.setExitRequestLimit / setMaxValidatorsPerReport
```
Pausing VEBO blocks `submitReportData`/`submitExitRequestsData` (every `whenResumed` ingest path) and `triggerExits`, but the VEDV proof and `getDeliveryTimestamp` reads stay available. WQ/TWG pause + NFT metadata + `WithdrawalVault` recovery admin: [`04`](./04-withdrawals.md#core-flows).

## Internal mechanics

**VEB hash registry.** `exitRequestsHash → {contractVersion, deliveredExitDataTimestamp}`. Consensus path (`submitReportData`) stores a delivered hash directly with `_getTime()`. Staged path requires a pre-committed (`contractVersion != 0`), not-yet-delivered (`timestamp == 0`) hash, consumes the limit, then marks delivered. `triggerExits` requires the hash already **delivered** and `exitDataIndexes` strictly increasing within range. VEDV reads `deliveredExitDataTimestamp` via `getDeliveryTimestamp` — the delivery time is the trust anchor for the delay clock.

**VEBO init / version gate.** `initialize(admin, consensus, consensusVersion, lastProcessingRefSlot, maxValidatorsPerRequest, maxExitRequestsLimit, exitsPerFrame, frameDurationInSec)` grants `DEFAULT_ADMIN_ROLE` to `admin` (role-admin for all VEB/VEBO roles), pauses infinitely, wires consensus and v2 rate params. `finalizeUpgrade_v2(...)` is the one-shot v1→v2 migrator; both route through `_updateContractVersion(2)` so each runs at most once.

**Exit-rate limit.** The sliding-window limit VEB consumes (`setExitRequestLimit`) and the one TWG consumes are each the packed `ExitLimitUtils` lib documented in [`04`](./04-withdrawals.md#internal-mechanics), applied to that contract's own slot under its own manager role — TWG's and VEB's budgets are independent; `type(uint256).max` = no throttle when unset. Separately, `setMaxValidatorsPerReport` requires a non-zero value (`ZeroArgument`) and caps each `submitExitRequestsData` payload — `requestsCount > maxValidatorsPerReport` ⇒ `TooManyExitRequestsInReport` (a per-report count cap, distinct from the sliding-window limit above).

## External interactions
```text
ValidatorsExitBus / ValidatorsExitBusOracle
  ← BaseOracle + HashConsensus (separate committee)            ← 00 (consensus model)
  ← submitReportData [SUBMIT_DATA_ROLE | consensus member] ; submitExitRequestsHash [SUBMIT_REPORT_HASH_ROLE]
  ← submitExitRequestsData / triggerExits [permissionless]
  → OracleReportSanityChecker.checkExitBusOracleReport         → 03 (damage bound)
  → emit ValidatorExitRequest (advisory — off-chain operators) ; → TWG.triggerFullWithdrawals   → 04

ValidatorExitDelayVerifier
  ← anyone (permissionless)
  → BEACON_ROOTS predeploy (EIP-4788)                          → R (proof model)
  → veb.unpackExitRequest / veb.getDeliveryTimestamp
  → StakingRouter.reportValidatorExitDelay                     → 02 (penalty applied in the module)
```

## Key constants

| Constant | Value | Purpose |
|---|---|---|
| `BEACON_ROOTS` | `0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02` | EIP-4788 timestamp→beacon-root oracle (VEDV) |
| `FAR_FUTURE_EPOCH` | `type(uint64).max` | "exit unset" invariant in the VEDV SSZ leaf |
| `DATA_FORMAT_LIST` | 1 | Only supported VEB packed-list format |
| `PACKED_REQUEST_LENGTH` / `PUBLIC_KEY_LENGTH` | 64 / 48 bytes | VEB request layout: 3+5+8+48 |

EIP-4788 / SSZ proof model (`historical_summaries` fallback, GIndex navigation, the `FAR_FUTURE_EPOCH` reconstruction) is distilled in [`R`](./R-consensus-proof-reference.md#distilled-external-specs); the EIP-7002 predeploy (`WITHDRAWAL_REQUEST`) the gateway calls is documented in [`04`](./04-withdrawals.md#key-constants).

## Source references

**Live source** (every symbol cited inline above resolves against these files):
- `contracts/0.8.9/oracle/`: `ValidatorsExitBus.sol` / `ValidatorsExitBusOracle.sol` (`triggerExits`/`getDeliveryTimestamp`/`unpackExitRequest`, hash registry, `DATA_FORMAT_LIST`/`PACKED_REQUEST_LENGTH`, `initialize`/`finalizeUpgrade_v2`). `contracts/0.8.25/ValidatorExitDelayVerifier.sol` (`BEACON_ROOTS`/`FAR_FUTURE_EPOCH`, `verify[Historical]ValidatorExitDelay`, `_getSecondsSinceExitIsEligible`).

**Official docs (docs/docs/):** `contracts/validators-exit-bus-oracle.md`, `contracts/validator-exit-delay-verifier.md`, `guides/oracle-spec/validator-exit-bus.md`, `guides/oracle-spec/penalties.md`.
