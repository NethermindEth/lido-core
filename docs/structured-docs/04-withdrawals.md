---
doc: "04"
title: Withdrawals
contracts: [WithdrawalQueue, WithdrawalQueueBase, WithdrawalQueueERC721, WithdrawalVault, WithdrawalVaultEIP7002, TriggerableWithdrawalsGateway, ExitLimitUtils]
prereqs: ["03"]
see_also: ["00","R","08","07"]
ssot_for: [withdrawal-finalize, finalize-burn-chain, triggerable-twg]
---
# 04 — Withdrawals

> The stETH→ETH withdrawal path: request, oracle-driven finalize + burn chain, claim, bunker mode, and the EIP-7002 triggerable-withdrawal gateway (TWG). Finalization is **driven by the oracle report** ([`03`](./03-oracle-accounting.md#core-flows) queues the burn and calls `WithdrawalQueue.finalize`). The validator-exit machinery (VEB / VEBO / VEDV) lives in [`08`](./08-exits.md#core-flows); EIP-7002 specs are distilled in [`R`](./R-consensus-proof-reference.md#distilled-external-specs). Roles cross-index in [`07`](./07-governance-permissions.md#contracts).

## Contracts

| Contract | File | Role |
|---|---|---|
| `WithdrawalQueue` | `0.8.9/WithdrawalQueue.sol` | stETH layer: request bounds, role gating, bunker coupling, claim payout. Inherits `WithdrawalQueueBase`. |
| `WithdrawalQueueBase` | `0.8.9/WithdrawalQueueBase.sol` | Queue arithmetic: cumulative totals, `calculateFinalizationBatches`/`prefinalize`/`_finalize`, checkpoints. |
| `WithdrawalQueueERC721` | `0.8.9/WithdrawalQueueERC721.sol` | Top layer; unstETH NFT (ERC-721/4906); deployed contract; exposes `finalize`. |
| `WithdrawalVault` | `0.8.9/WithdrawalVault.sol` | Lands CL-WC ETH, pushes to `Lido` on report; thin EIP-7002 caller; permissionless recovery to `TREASURY`. |
| `WithdrawalVaultEIP7002` | `0.8.9/WithdrawalVaultEIP7002.sol` | Base of `WithdrawalVault`; raw EIP-7002 predeploy calls + fee read. |
| `TriggerableWithdrawalsGateway` (TWG) | `0.8.9/TriggerableWithdrawalsGateway.sol` | Role-gated, rate-limited EIP-7002 full-exit entrypoint; fee/refund; notifies `StakingRouter`. |
| `ExitLimitUtils` | `0.8.9/lib/ExitLimitUtils.sol` | Sliding-window exit-rate-limit lib (packed `uint32` state); `using` by TWG and VEB (`08`). |
| `Lido`/`Burner`/`AccountingOracle`/`StakingRouter` | (boundary) | finalize caller + share-burn sink + bunker report source + exit-triggered sink. |

## Core flows

### 1. Request (permissionless)
All four entrypoints (`requestWithdrawals`, `*WithPermit`, `*WstETH`, `*WstETHWithPermit`) gate on `_checkResumed()` only.
```text
user → WithdrawalQueueERC721.requestWithdrawals(amounts[], owner)   // _checkResumed
  → per amount: require MIN_STETH_WITHDRAWAL_AMOUNT <= amount <= MAX_STETH_WITHDRAWAL_AMOUNT  // 100 wei .. 1000 ether
  → STETH.transferFrom(msg.sender, this, amount)   // wstETH path unwraps first
  → shares = STETH.getSharesByPooledEth(amount); _enqueue: queue[++lastRequestId] = WithdrawalRequest{...}; _mint(owner, requestId)
```
`cumulativeStETH`/`cumulativeShares` are **running totals**, so finalization reads a contiguous batch's exact stETH/shares in O(1) by differencing endpoints. `reportTimestamp` (last report seen at enqueue) is the batch-grouping key (flow 2).

### 2. Finalize on oracle report — the burn chain (LOAD-BEARING)
The daemon pre-computes the optimal batch off-chain via `calculateFinalizationBatches(_maxShareRate, _maxTimestamp, _maxRequestsPerCall, state)` and packs the request ids into the report's `withdrawalFinalizationBatches` field. On-chain, [`03 Accounting`](./03-oracle-accounting.md#core-flows) re-derives the numbers and drives the burn **before** finalize:
```text
Accounting._applyOracleReportContext
  → WithdrawalQueue.prefinalize(batches[], simulatedShareRate) → (ethToLock, sharesToBurn)   // on-chain re-derivation
  → Burner.requestBurnShares(withdrawalQueue, sharesToBurn)        // EXT: queues the WQ-finalized share burn
  → Burner.commitSharesToBurn(totalAggregate)                     // EXT: commits the AGGREGATE (WQ + rebase) → Lido.burnShares
  → Lido.collectRewardsAndProcessWithdrawals(wvTransfer, elTransfer, ethToLock, simulatedShareRate)
      → WithdrawalVault.withdrawWithdrawals(wvTransfer)            // EXT: pulls CL-withdrawal ETH into buffer
      → WithdrawalQueueERC721.finalize{value: ethToLock}(lastReqId, simulatedShareRate)   // _checkResumed + FINALIZE_ROLE(=Lido)
External: Lido, Burner, WithdrawalVault.
```
Key ordering / invariants:
- The **burn is requested and committed against the WQ's own stETH balance by Accounting** (not inside `finalize`). `WQ.finalize` only checkpoints (appends exactly one `Checkpoint`, bumping `lastCheckpointIndex`), locks ETH, advances `lastFinalizedRequestId`; it never touches shares. Aggregate `commitSharesToBurn` folds WQ shares with the rebase burn so `Lido.burnShares` runs once.
- `prefinalize` clamps each batch: if `batchShareRate > _maxShareRate` the ether is **discounted** to `shares × _maxShareRate / E27_PRECISION_BASE`; else nominal `stETH`. There is **no on-chain bunker branch** — the conservative bunker rate is realized purely by the daemon passing a lower `_maxShareRate` (the bunker flag in flow 4 only steers that off-chain choice).
- `_finalize` reverts `TooMuchEtherToFinalize` if `msg.value` exceeds the batch's `cumulativeStETH` delta — ETH locked can never exceed stETH owed.
- Rewards accrued while stETH sat queued are **not** re-attributed; they burn with the shares ("no rewards during exit").
- **Stale-rate guard:** `requestTimestampMargin` is an on-chain `OracleReportSanityChecker.LimitsList` field (governance-set via `setRequestTimestampMargin` under `REQUEST_TIMESTAMP_MARGIN_MANAGER_ROLE`), enforced by `checkWithdrawalQueueOracleReport` (the last finalizable request must be at least that old); the daemon mirrors it into `_maxTimestamp` so only matured requests enter a batch — `calculateFinalizationBatches` breaks on `request.timestamp > _maxTimestamp`.
- **`MAX_BATCHES_LENGTH = 36`** hard-caps the batches array (`break` on the 37th distinct batch) so on-chain `prefinalize`/`finalize` arrays stay gas-bounded. Same-report requests (equal `reportTimestamp`) collapse into one batch despite 1-2 wei rate drift.

### 3. Claim (caller-identity)
```text
NFT owner → WithdrawalQueue.claimWithdrawal(id) / claimWithdrawals(ids[], hints[]) / claimWithdrawalsTo(ids[], hints[], recipient)
  → require id <= lastFinalizedRequestId, not claimed, ownerOf(id)==msg.sender
  → eth = _calculateClaimableEther: batchRate>checkpoint.maxShareRate ? shares*maxShareRate/E27 : cumulativeStETH delta
  → claimed=true; _burn(id); lockedEtherAmount -= eth; low-level call eth → recipient
```
Client-supplied `hints` are range-checked in `_calculateClaimableEther` (reverting `InvalidHint`); `_findCheckpointHint` (binary search over `[1, lastCheckpointIndex]`) is the hint generator used by `findCheckpointHints` and by the no-hint `claimWithdrawal` (which runs the search itself, O(log n), costing more gas). 1-2 wei dust accrues per request from `/E27_PRECISION_BASE` rounding. Claiming stays available while paused.

### 4. Bunker mode (oracle-set sentinel)
`AccountingOracle` → `WithdrawalQueue.onOracleReport(isBunkerNow, bunkerStartTimestamp, currentReportTimestamp)`, gated `_checkRole(ORACLE_ROLE)`. Entering sets `BUNKER_MODE_SINCE_TIMESTAMP_POSITION` to `_bunkerStartTimestamp` (oracle-supplied activation time, NOT `block.timestamp`); exiting writes sentinel `BUNKER_MODE_DISABLED_TIMESTAMP = type(uint256).max`. `isBunkerModeActive()` is `bunkerModeSinceTimestamp() < BUNKER_MODE_DISABLED_TIMESTAMP`. The flag is opaque on-chain — only `AccountingOracle` writes it; the daemon derives `isBunkerMode` from LIP-23 heuristics. Purpose: while the protocol carries an unrealized CL loss, finalizing at the normal rate would let exiting holders claim at the pre-loss rate and leave the shortfall to remaining holders; the daemon's lower `_maxShareRate` makes exiting holders absorb their pro-rata share of the loss. The realization is entirely off-chain — no function branches on `isBunkerModeActive()`; `_finalize`/`_calculateClaimableEther` apply whatever rate the report carries (flow 2).

### 5. CL withdrawals → WithdrawalVault → Lido
`0x01`/`0x02` validators push balances to `WithdrawalVault` (proxied, no AccessControl). On report, `Lido.collectRewardsAndProcessWithdrawals` → `WithdrawalVault.withdrawWithdrawals(amount)` (reverts `NotLido()` unless `msg.sender == LIDO`; `ZeroAmount`/`NotEnoughEther`) → `Lido.receiveWithdrawals{value}`. Amount bounded by the report's `withdrawalsVaultTransfer` (capped upstream by sanity-checker positive-rebase smoothing → [`03`](./03-oracle-accounting.md#core-flows)).

### 6. Triggerable EIP-7002 forced exit — TWG → predeploy (LOAD-BEARING)
Reached from `ValidatorsExitBus.triggerExits` (the VEB-side packing + hash gate lives in [`08`](./08-exits.md#core-flows)); the gateway owns the fee/refund + predeploy mechanics:
```text
TWG.triggerFullWithdrawals{value}(validatorsData[], refundRecipient, EXIT_TYPE)  // onlyRole ADD_FULL_WITHDRAWAL_REQUEST_ROLE (VEB), whenResumed, preservesEthBalance
  → _consumeExitRequestLimit(count)                          // ExitLimitUtils sliding window; revert ExitRequestsLimitExceeded
  → fee = WithdrawalVault.getWithdrawalRequestFee(); totalFee = count*fee
  → refund = _checkFee(totalFee)                            // TWG: revert InsufficientFee if msg.value < totalFee
  → WithdrawalVault.addWithdrawalRequests{value: totalFee}(pubkeys[], new uint64[](count))  // amounts all 0 ⇒ full exit
      → per pubkey: WITHDRAWAL_REQUEST.call{value: fee}(pubkey ‖ uint64(0))  // EXT: EIP-7002 predeploy
  → StakingRouter.onValidatorExitTriggered(validatorsData[], fee, exitType)  // EXT: single batch, per-validator fee
  → _refundFee(refund, refundRecipient)                     // recipient==0 ⇒ msg.sender; revert FeeRefundFailed
External: EIP-7002 predeploy, StakingRouter.
```
Why two `_checkFee`s differ: TWG forwards **exactly** `totalFee` and refunds the surplus (`msg.value >= totalFee`), but `WithdrawalVaultEIP7002._checkFee` requires `msg.value == fee` **exactly** (`IncorrectFee`) — the vault tolerates no slack, so TWG sizes the forwarded value precisely. `preservesEthBalance` on both asserts each contract's own balance is unchanged, so no ETH is stranded or skimmed. `onValidatorExitTriggered` runs only after the predeploy calls succeed (atomic tx), keeping module exit-accounting 1:1 with accepted EIP-7002 requests. `ADD_FULL_WITHDRAWAL_REQUEST_ROLE` is held by VEBO (granted at deploy), reached via VEB ([`08`](./08-exits.md#core-flows)); the `StakingVault` path uses a **separate** `TriggerableWithdrawals` lib, NOT this gateway ([`06`](./06-vaults.md#core-flows)).

### 7. Admin — pause, resume, rate-limit, NFT metadata, recovery
```text
PAUSE_ROLE  → WQ.pauseFor/pauseUntil ; TWG.pauseFor/pauseUntil          (GateSeal can pause)
RESUME_ROLE → WQ.resume ; TWG.resume                                    (WQ deploys paused; TWG deploys RESUMED)
TW_EXIT_LIMIT_MANAGER_ROLE → TWG.setExitRequestLimit(max, exitsPerFrame, frameDurationInSec)
MANAGE_TOKEN_URI_ROLE      → WQ721.setBaseURI / setNFTDescriptorAddress
anyone → WithdrawalVault.recoverERC20/recoverERC721 → always to immutable TREASURY   (no ETH-recovery path)
```
Pausing blocks every `whenResumed`/`_checkResumed` entrypoint (request, finalize, TWG trigger) but **not** claiming. unstETH transfers are owner/approved-gated; `_transfer` reverts `RequestAlreadyClaimed` on claimed ids (finalized-but-unclaimed stay transferable). `WithdrawalVault` has no ETH-recovery path (fee flow asserts `preservesEthBalance`), so CL-withdrawal ETH cannot be swept. VEBO / VEB pause + exit-rate-limit admin: [`08`](./08-exits.md#core-flows).

## Internal mechanics

**WQ storage (unstructured slots).** `queue: id→WithdrawalRequest` (cumulative, not delta); `checkpoints: index→Checkpoint(fromRequestId, maxShareRate)`; scalars `lastRequestId`, `lastFinalizedRequestId`, `lastCheckpointIndex`, `lockedEtherAmount`. Id 0 is a sentinel (`claimed=true`, zero amounts) so endpoint differencing works for the first batch.
```solidity
struct WithdrawalRequest {        // WithdrawalQueueBase
    uint128 cumulativeStETH;      // running sum
    uint128 cumulativeShares;     // running sum
    address owner;
    uint40 timestamp;             // enqueue time (maturity gate in calculateFinalizationBatches)
    bool claimed;                 // one-way
    uint40 reportTimestamp;       // last report seen at enqueue — batch grouping key
}
```
**CEI / rounding / ordering hazards.** Burn-before-finalize (flow 2): shares are committed for burn though `finalize` runs later in the same tx — atomicity holds, but treat `prefinalize` numbers and `finalize`'s `msg.value` as a coupled pair (mismatch reverts `TooMuchEtherToFinalize`). Claim is CEI-clean (state flipped before the low-level ETH `call`), so no reentrancy lever (`claimed`/`_burn` precede the transfer). Discount math floors via integer `/E27_PRECISION_BASE` ⇒ protocol-favouring 1-2 wei dust.

**ExitLimitUtils sliding window.** Packed `ExitRequestLimitData` of five `uint32`s: `maxExitRequestsLimit`, `prevExitRequestsLimit`, `prevTimestamp`, `frameDurationInSec`, `exitsPerFrame`. `calculateCurrentExitLimit(now)` restores `prevExitRequestsLimit + framesPassed * exitsPerFrame` since `prevTimestamp`, capped at `max`; returns `prev` unchanged inside a frame or when `exitsPerFrame==0`. `updatePrevExitLimit` advances `prevTimestamp` only by whole frames (`passedTime -= passedTime % frameDuration`) so sub-frame time is not lost. `isExitLimitSet()` is `max != 0`; when unset, TWG/VEB treat the limit as `type(uint256).max` (no throttle). `setExitLimits` carries forward `exitsUsed = max - currentLimit`. Reverts: `TooLargeMaxExitRequestsLimit`/`TooLargeFrameDuration` (uint32), `TooLargeExitsPerFrame` (> max), `ZeroFrameDuration`. TWG and VEB each hold their own `ExitRequestLimitData` in a distinct storage slot, set by distinct roles (`TW_EXIT_LIMIT_MANAGER_ROLE` vs `EXIT_REQUEST_LIMIT_MANAGER_ROLE`); the two budgets are independent — consuming one never affects the other.

## External interactions
```text
WithdrawalQueueERC721 / WithdrawalQueue / Base
  ← user             requestWithdrawals* [_checkResumed]; claimWithdrawal* [ownerOf]; ERC721 transfer [owner/approved]
  ← Lido             FINALIZE_ROLE → finalize (payable, locks ETH)
  ← AccountingOracle ORACLE_ROLE → onOracleReport (bunker flag)
  ← MANAGE_TOKEN_URI_ROLE → setBaseURI / setNFTDescriptorAddress
  → STETH            transferFrom, getSharesByPooledEth
  → ETH recipient    low-level call on claim
  (share burn is requested by Accounting on Burner — see 03; finalize itself does NOT burn)

WithdrawalVault (+ WithdrawalVaultEIP7002)   — no AccessControl; caller-identity only
  ← validators       CL 0x01/0x02 push ETH
  ← Lido             withdrawWithdrawals (msg.sender==LIDO) → Lido.receiveWithdrawals
  ← TWG              addWithdrawalRequests (msg.sender==TRIGGERABLE_WITHDRAWALS_GATEWAY)
  ← anyone           recoverERC20/recoverERC721 → TREASURY
  → WITHDRAWAL_REQUEST predeploy (full exit; no consolidation path here — that is the vault path in 06)

TriggerableWithdrawalsGateway
  ← ADD_FULL_WITHDRAWAL_REQUEST_ROLE → triggerFullWithdrawals ; TW_EXIT_LIMIT_MANAGER_ROLE → setExitRequestLimit
  ← VEB              triggerExits forwards here                  ← 08
  → WithdrawalVault.addWithdrawalRequests ; StakingRouter.onValidatorExitTriggered
```

## Key constants

| Constant | Value | Purpose |
|---|---|---|
| `MIN_STETH_WITHDRAWAL_AMOUNT` | 100 wei | Lower bound per WQ request |
| `MAX_STETH_WITHDRAWAL_AMOUNT` | 1000 ether | Upper bound per WQ request (anti-clog) |
| `MAX_BATCHES_LENGTH` | 36 | Hard cap on finalization batches per report (gas bound) |
| `E27_PRECISION_BASE` | 1e27 | Share-rate fixed-point base for discount math |
| `BUNKER_MODE_DISABLED_TIMESTAMP` | `type(uint256).max` | Turbo-mode sentinel for `bunkerModeSinceTimestamp` |
| `WITHDRAWAL_REQUEST` | `0x00000961Ef480Eb55e80D19ad83579A64c007002` | EIP-7002 predeploy; `staticcall("")` → uint256 fee; `call{value:fee}(pubkey‖uint64 amount)`; amount=0 ⇒ full exit; `msg.sender` must hold the 0x01 source (the vault) |
| `VERSION` (TWG) | 1 | Gateway version |

EIP-7002 spec (56-byte `48-byte pubkey ‖ 8-byte uint64 amount` request, `amount == 0` = full exit, dynamic `staticcall("")` fee, `count × fee` with surplus refund) is distilled in [`R`](./R-consensus-proof-reference.md#distilled-external-specs). Lido-specific load-bearing rule: the predeploy enforces `msg.sender == the 0x01 WC source`, which is **why the WC-holding `WithdrawalVault` must be the caller** — TWG routes the fee through the vault.

## Source references

**Live source** (every symbol cited inline above resolves against these files):
- `contracts/0.8.9/`: `WithdrawalQueue.sol` / `WithdrawalQueueBase.sol` / `WithdrawalQueueERC721.sol` (request/finalize/claim, `calculateFinalizationBatches`/`prefinalize`/`_finalize`, `onOracleReport`/bunker sentinel, `MAX_BATCHES_LENGTH`/`E27_PRECISION_BASE`); `WithdrawalVault.sol` / `WithdrawalVaultEIP7002.sol` (`withdrawWithdrawals`/`NotLido`, `WITHDRAWAL_REQUEST`, `_checkFee == fee`); `TriggerableWithdrawalsGateway.sol` (`triggerFullWithdrawals`, `_checkFee >=`, `preservesEthBalance`); `lib/ExitLimitUtils.sol` (sliding window).

**Official docs (context/docs/docs/):** `contracts/withdrawal-queue-erc721.md`, `contracts/withdrawal-vault.md`, `contracts/triggerable-withdrawals-gateway.md`.
