---
doc: "R"
title: Consensus & Proof Reference
contracts: [SSZ, GIndex]
prereqs: []
see_also: ["04","06","08"]
ssot_for: [ssz-gindex, proof-model, eip-7002, eip-7251, eip-4788, withdrawal-credentials, cl-request-processing]
---
# R — Consensus and Proof Reference

> Shared consensus-layer reference. The in-repo proof libraries `SSZ`/`GIndex` and the distilled EIP facts
> (7002/7251/4788/6110, WC types, balance caps) live here ONCE; the consuming verifiers carry a one-line
> pointer + their own load-bearing constant. Consumed by [`06`](./06-vaults.md#core-flows) (PDG `CLProofVerifier`,
> EIP-7002/7251 vault exits), [`08`](./08-exits.md#core-flows) (`ValidatorExitDelayVerifier`), and [`04`](./04-withdrawals.md#core-flows) (`WithdrawalVaultEIP7002`).

## Contracts

| Contract | File | Role |
|---|---|---|
| `SSZ` | `common/lib/SSZ.sol` | SSZ Merkle hashing (`hashTreeRoot` for `Validator` and `BeaconBlockHeader`) + branch verification (`verifyProof`) via the SHA-256 precompile (address `0x02`). |
| `GIndex` | `common/lib/GIndex.sol` | `type GIndex is bytes32` generalized-index math (`pack`/`index`/`width`/`pow`/`shr`/`shl`/`concat`) that tells `verifyProof` the branch shape (tree depth + leaf position). |
| `BeaconBlockHeader`, `Validator` | `common/lib/BeaconTypes.sol` | (boundary) leaf containers `SSZ.hashTreeRoot` serializes. |
| `CLProofVerifier`, `PredepositGuarantee`, `ValidatorExitDelayVerifier` | (boundary, owned by [`06`](./06-vaults.md#contracts)/[`08`](./08-exits.md#contracts)) | the three consumers that import `SSZ`/`GIndex` and supply the fork-pivot GIndices + EIP-4788 root. |

`SSZ`/`GIndex` are the proof hub for the whole repo: a flaw in either breaks vault deposit security (PDG) AND
exit-penalty verification (VEDV) simultaneously — the load-bearing reason both proof surfaces share one library layer.

## Core flows

### 1. SSZ branch verification (the proof core)

`SSZ.verifyProof(bytes32[] proof, bytes32 root, bytes32 leaf, GIndex gI)` walks a Merkle branch from `leaf` up
to `root`; **`gI` is the trusted description of where the leaf sits** in the tree. The walk reads the depth/position
from the GIndex and folds each `proof[i]` into `leaf` with SHA-256.

```text
verifyProof(proof, root, leaf, gI)
  -> index = gI.index()                       // generalized index (gI >> 8)
  -> require proof.length != 0                 // else revert InvalidProof
  -> loop over proof[i]:
       scratch = (index & 1) ? 0x20 : 0x00     // sibling side from current bit
       index >>= 1
       require index != 0                      // else revert BranchHasExtraItem (proof too long for gI)
       leaf = sha256(ordered(leaf, proof[i]))  // EXT: precompile 0x02
  -> require index == 1                         // else revert BranchHasMissingItem (proof too short for gI)
  -> require leaf == root                       // else revert InvalidProof
External: SHA-256 precompile (0x02) only; no storage, view-only.
```

Why the index-consume invariant matters: the loop shifts `index` right once per proof element. If the proof is
*longer* than the GIndex depth, `index` hits 0 mid-walk → `BranchHasExtraItem`; if *shorter*, `index` ends > 1 →
`BranchHasMissingItem`. So the GIndex depth and the proof length must agree exactly — a caller cannot smuggle a
valid-looking branch at the wrong depth. The final `index == 1` check confirms the walk reached the tree root
position, and `leaf == root` confirms it reconstructed the committed root. There is no length sanity-check on the
GIndex beyond this, so the **GIndex is fully trusted input** — it is supplied by the verifier contract's immutables,
never by the proof submitter (see flow 3).

### 2. Leaf reconstruction — hashTreeRoot

The leaf fed to `verifyProof` is normally a container root the contract computes itself, so a submitter cannot
forge it. `hashTreeRoot(Validator)` and `hashTreeRoot(BeaconBlockHeader)` SSZ-merkleize their fields (`Validator` has 8, `BeaconBlockHeader` 5)
(pad to 8 leaves, pairwise SHA-256 up the balanced tree). Two endianness/encoding gotchas the implementation must
match the CL exactly:

- **uint64/bool fields are little-endian** (`toLittleEndian`): `slot`, `proposerIndex`, `effectiveBalance`,
  `slashed`, the four epoch fields. A big-endian transcription would silently produce a wrong-but-deterministic
  root that fails `leaf == root`.
- **`pubkey` is itself a merkleized leaf, not raw bytes**: the 48-byte pubkey is copied into a 64-byte scratch
  word, the trailing 16 bytes zeroed, then hashed once with SHA-256 — i.e. `sha256(pubkey ++ 16 zero bytes)`.
  Hashing the raw 48 bytes would be wrong.

`Validator` field order is `(pubkey, withdrawalCredentials, effectiveBalance, slashed, activationEligibilityEpoch,
activationEpoch, exitEpoch, withdrawableEpoch)`; `BeaconBlockHeader` is `(slot, proposerIndex, parentRoot,
stateRoot, bodyRoot)`. VEDV calls the full `hashTreeRoot(Validator)` (with `exitEpoch` pinned to `FAR_FUTURE_EPOCH`)
and `hashTreeRoot(BeaconBlockHeader)`. PDG (`CLProofVerifier`) does NOT merkleize the whole `Validator`: it proves
the `(pubkey, withdrawalCredentials)` parent subtree node via `BLS12_381.sha256Pair(BLS12_381.pubkeyRoot(pubkey), wc)` and binds that
deeper with a concatenated GIndex (flow 4).

### 3. GIndex navigation and the fork-pivot

A `GIndex` packs `(generalized_index << 8) | depth` into a `bytes32` (`pack`). `index()` recovers the gindex,
`pow()` the depth, `width() = 1 << pow`. Two operations build the branch shape:

- **`shr(offset)`** — sibling to the right within the same level: used to jump from "first validator" to
  validator N. `_getValidatorGI(offset, slot)` does exactly `firstValidatorGI.shr(validatorIndex)`.
- **`concat(parent, child)`** — splices a child sub-tree GIndex under a parent GIndex (used by `CLProofVerifier`
  to traverse block-header → state-root → validators-list → validator → `(pubkey, WC)` parent as one combined index).

**Fork-pivot PREV/CURR (load-bearing).** Container layouts shift across hard forks, which moves the generalized
index of "first validator" / "first historical_summary" / "first block-root-in-summary". Each verifier holds a
PREV and a CURR immutable for every such anchor and a `PIVOT_SLOT`, and selects by the *proven slot*:

```text
_getValidatorGI(offset, provenSlot):
  base = provenSlot < PIVOT_SLOT ? GI_FIRST_VALIDATOR_PREV : GI_FIRST_VALIDATOR_CURR
  return base.shr(offset)             // offset = validator index
```

Gotcha: proving a post-fork block with the PREV GIndex (or vice-versa) verifies the leaf against the *wrong tree
position*. The slot comparison is the only thing binding the GIndex to the layout, so `PIVOT_SLOT` and the four
immutable GIndices are deployment-critical trusted parameters. VEDV carries the same PREV/CURR pairing for the
`historical_summaries` path (`GI_FIRST_HISTORICAL_SUMMARY_*`, `GI_FIRST_BLOCK_ROOT_IN_SUMMARY_*`).

### 4. EIP-4788 beacon-root anchoring (the trust anchor)

The whole proof chain bottoms out in a beacon block root that the EL trusts because EIP-4788 exposes it. The two
consumers anchor differently — read the source, not a single merged path: VEDV binds the header by direct equality,
PDG binds it inside one combined-GIndex proof.

```text
shared step 0: root = BEACON_ROOTS.staticcall(abi.encode(timestamp))   // EXT: 0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02
                 -> revert RootNotFound if empty (slot outside the 8191-slot ring buffer)

VEDV (ValidatorExitDelayVerifier):
  1. require root == header.hashTreeRoot()                       // _verifyBeaconBlockRoot: equality, NOT verifyProof
  2. SSZ.verifyProof(validatorProof, header.stateRoot,
                     validator.hashTreeRoot(),                    // leaf has exitEpoch hard-coded to FAR_FUTURE_EPOCH
                     _getValidatorGI(validatorIndex, header.slot))

PDG (CLProofVerifier):
  1. leaf = BLS12_381.sha256Pair(BLS12_381.pubkeyRoot(pubkey), withdrawalCredentials) // parent(pubkey, wc) subtree node
  2. SSZ.verifyProof(proof, root, leaf,                           // ONE proof straight to the beacon root
                     concat(GI_STATE_ROOT, concat(_getValidatorGI(idx, slot), GI_PUBKEY_WC_PARENT)))
External: BEACON_ROOTS (0x..0Beac02) staticcall + SHA-256 (0x02). View-only.
```

So VEDV reads `exitEpoch` (by reconstructing the validator leaf), PDG reads `withdrawalCredentials` (folded into the
proven leaf). Before the proven `slot` selects the PREV/CURR fork GIndex (the `PIVOT_SLOT` choice in flow 3),
`CLProofVerifier._verifySlot` binds the user-supplied `slot`/`proposerIndex` to a Merkle branch (`proof[len-2]` via
`SLOT_PROPOSER_PARENT_PROOF_OFFSET`, revert `InvalidSlot`) — blocking a cross-slot proof replay that would let a
caller pick a favorable fork GIndex. This `_verifySlot` step is PDG-only; VEDV shares the `PIVOT_SLOT` fork-GIndex
selection (`_getValidatorGI`) but instead pins `slot`/`proposerIndex` via full-header `hashTreeRoot` equality
(`_verifyBeaconBlockRoot`), so it neither imports `CLProofVerifier` nor calls `_verifySlot`. For slots **older than the 8191-slot buffer**, step 0 cannot resolve the timestamp; the verifier
instead proves the old block root through the beacon state's `historical_summaries` accumulator (Capella+): with
`targetSlot` and `recentSlot`, compute `summaryIndex = (targetSlot − CAPELLA_SLOT) / SLOTS_PER_HISTORICAL_ROOT` and
`rootIndex = targetSlot % SLOTS_PER_HISTORICAL_ROOT`, then `verifyProof` the old header against the recent block's
`stateRoot` using `_getHistoricalBlockRootGI` (`GI_FIRST_HISTORICAL_SUMMARY_*.shr(summaryIndex)` then
`.concat(GI_FIRST_BLOCK_ROOT_IN_SUMMARY_*).shr(rootIndex)`, the summary anchor PREV/CURR-selected by `recentSlot`, the block-root anchor by the summary's creation slot). The recent anchor (a
still-buffered block) is the EIP-4788 root; everything else hangs off it by SSZ branches.

## Distilled external specs

Lido-relevant facts only.

- **EIP-7002 (triggerable withdrawals/exits).** Request = **56 bytes** (`WITHDRAWAL_REQUEST_CALLDATA_LENGTH`) =
  48-byte pubkey ++ big-endian `uint64` amount (`WITHDRAWAL_AMOUNT_LENGTH = 8`); `amount = 0` ⇒ full exit (no
  named constant in source — the source comment is "withdrawal amount = 0"). Submitted to predeploy `…007002`
  (`WITHDRAWAL_REQUEST` in both `WithdrawalVaultEIP7002` and `common/lib/TriggerableWithdrawals`). Fee is dynamic
  (EIP-1559-style on a queue-excess counter): fee getter is `predeploy.staticcall("")` returning a `uint256`;
  the wrapper sends `call{value: fee}(request)` per pubkey and requires `msg.value == totalFee` exactly
  (`WithdrawalVaultEIP7002._checkFee`, revert `IncorrectFee`); the `msg.value − totalFee` refund is caller-side
  (`TriggerableWithdrawalsGateway`/`StakingVault`), not in the EIP-7002 encoder. The system contract uses the
  caller (`msg.sender`) as the withdrawal-request `source_address`; `process_withdrawal_request` then requires
  `has_execution_withdrawal_credential` (`0x01` or `0x02`) and `withdrawal_credentials[12:] == source_address`, so the
  calling contract must hold the validator's execution-withdrawal credential — for a V3 `StakingVault`, its `0x02` WC.
  *(Not covered here: in-state queue layout, dequeue/excess-update/count-reset helpers, synthetic
  deployment blob, 30M system-call gas, EIP-7685 wrapping.)*
- **EIP-7251 (consolidation).** Predeploy `…007251` (`CONSOLIDATION_REQUEST_PREDEPLOY_ADDRESS`). Calldata = **96
  bytes** = source pubkey ‖ target pubkey (`2 × 48`). Merges a source validator into a `0x02` (compounding) target;
  switching `0x01 → 0x02` raises a validator's effective-balance cap. Same staticcall("") fee-getter shape as 7002.
  In-repo encoder is `ValidatorConsolidationRequests` (Vault CLI; holds no funds, mutates no state).
- **EIP-4788 (beacon roots).** `BEACON_ROOTS` predeploy `0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02`
  (identical constant in `CLProofVerifier` and `ValidatorExitDelayVerifier`). Timestamp-keyed **8191-slot ring
  buffer** (`HISTORY_BUFFER_LENGTH`, prime by design) of beacon block roots; `staticcall(abi.encode(timestamp))` returns the root or empty. Slots older than
  the buffer use the `historical_summaries` fallback (flow 4).
- **Withdrawal-credential types.** `0x00` BLS (legacy, not used for new Lido validators); `0x01` Eth1-address
  (`0x01 ‖ 11 zero bytes ‖ 20-byte addr`, Core Pool → Lido `WithdrawalVault`, CL-spec cap MIN_ACTIVATION_BALANCE =
  32 ETH); `0x02` compounding (Electra, V3 `StakingVault` = `0x02 ‖ 11 zero bytes ‖ address(this)`, CL-spec cap
  MAX_EFFECTIVE_BALANCE_ELECTRA = 2048 ETH, required for 7251). Neither cap is a *named* in-repo constant:
  `MIN_ACTIVATION_BALANCE` appears only in a `Dashboard` doc-comment, and `2048` only as the `2048 ether` operand
  of `PredepositGuarantee.MAX_TOPUP_AMOUNT` (= 2048 − 31 − 1 = 2016 ETH top-up headroom, not the cap itself). **Invariant:** a validator never
  requested-and-initiated to exit has `exitEpoch == FAR_FUTURE_EPOCH` (`type(uint64).max`) — the property VEDV
  proves (the leaf is reconstructed with `exitEpoch` hard-coded to `FAR_FUTURE_EPOCH`). On exit, `withdrawableEpoch
  = exitEpoch + MIN_VALIDATOR_WITHDRAWABILITY_DELAY` (a fixed CL-spec offset).
- **EIP-6110 (deposit requests).** EL deposit-request → validator activation is **not instantaneous**: it enters
  the activation queue and is churn-limited (`get_activation_exit_churn_limit`), so PDG's WC proof can predate
  activation.

*(Out of in-repo scope: Pectra committee/attestation EIP-7549, sync-committee, blob EIP-7691,
proposer-selection (`get_beacon_proposer_index`), `process_slashings`, full `BeaconState`/`BeaconBlockBody` dumps, engine APIs. The CL request-processing model — `process_withdrawal_request` / `process_consolidation_request` /
`process_deposit_request` and the churn/queue/sweep mechanics — is detailed below in
[Consensus-layer request processing](#consensus-layer-request-processing-the-seam).)*

## Consensus-layer request processing (the seam)

The in-scope code submits EL-triggered requests — EIP-7002 withdrawals/exits (`TriggerableWithdrawals`,
`WithdrawalVaultEIP7002`), EIP-7251 consolidations (`ValidatorConsolidationRequests`), EIP-6110 deposits (the
PDG predeposit → activation path) — and proves CL state (`CLProofVerifier`, `ValidatorExitDelayVerifier`). A
successful on-chain submission only means the request was encoded and the fee was paid; the consensus layer then
runs `process_*_request` against live beacon state, applying a validation ladder, caps, and queue interactions.
Each ladder step is a **silent skip / cap / queue**: the EL submission returns success and the fee is spent, but
the resulting CL state change may be nothing, or a capped/queued amount that differs from the requested one. The
blocks below distill that model as the CL behavior of each request.

### EIP-7002 — `process_withdrawal_request`

`amount` is a big-endian `uint64`; `amount == FULL_EXIT_REQUEST_AMOUNT (== 0)` marks a full exit, any
`amount > 0` a partial withdrawal. Each guard below is a silent `return` — no state change, and no revert reaches
the EL caller:

```text
1. pending_partial_withdrawals queue full (len == PENDING_PARTIAL_WITHDRAWALS_LIMIT) ∧ not a full-exit  -> skip
2. request pubkey is not a known validator                                                              -> skip
3. !has_execution_withdrawal_credential(validator)  (WC is neither 0x01 nor 0x02)                        -> skip
   ∨ validator.withdrawal_credentials[12:] != request.source_address                                     -> skip
4. validator not active (is_active_validator false)                                                      -> skip
5. validator.exit_epoch != FAR_FUTURE_EPOCH  (an exit was already initiated)                             -> skip
6. current_epoch < validator.activation_epoch + SHARD_COMMITTEE_PERIOD  (validator too young)            -> skip
```

Past the ladder, behavior splits on `amount`:

- **Full exit** (`amount == 0`): initiates the exit **only if `get_pending_balance_to_withdraw(index) == 0`**; if
  the validator has any pending partial withdrawal queued, the full-exit request is consumed with no effect.
- **Partial** (`amount > 0`): additionally requires `has_compounding_withdrawal_credential` (a `0x01` validator
  can never partial-withdraw via 7002), `effective_balance >= MIN_ACTIVATION_BALANCE`, and
  `balance > MIN_ACTIVATION_BALANCE + pending_balance_to_withdraw`; if any of these does not hold, the request is
  consumed with no effect. When all hold, the queued amount is **capped** at
  `to_withdraw = min(amount, balance − MIN_ACTIVATION_BALANCE − pending_balance_to_withdraw)`, so the queued
  withdrawal can be smaller than `amount`; its `withdrawable_epoch` is churn-scheduled
  (`compute_exit_epoch_and_update_churn(to_withdraw) + MIN_VALIDATOR_WITHDRAWABILITY_DELAY`).

When the queued partial is later swept (`get_pending_partial_withdrawals`), it is **re-capped at withdrawal time**
to `min(balance − MIN_ACTIVATION_BALANCE, amount)` and is paid only while the validator still satisfies
`is_eligible_for_partial_withdrawals` (`exit_epoch == FAR_FUTURE_EPOCH ∧ effective_balance >= MIN_ACTIVATION_BALANCE
∧ balance > MIN_ACTIVATION_BALANCE`); otherwise that queued entry is skipped for the sweep.

### EIP-7251 — `process_consolidation_request` / `is_valid_switch_to_compounding_request`

The request carries `source_pubkey ‖ target_pubkey`. There are two distinct paths:

- **Switch-to-compounding** (`source_pubkey == target_pubkey`): valid (per
  `is_valid_switch_to_compounding_request`) only if the source pubkey is known,
  `withdrawal_credentials[12:] == source_address`, the source has `has_eth1_withdrawal_credential` (it is `0x01`),
  the source is active, and its `exit_epoch == FAR_FUTURE_EPOCH`. When valid, `switch_to_compounding_validator`
  flips the WC prefix to `COMPOUNDING_WITHDRAWAL_PREFIX (0x02)` and `queue_excess_active_balance` queues
  `balance − MIN_ACTIVATION_BALANCE` (when positive) as a pending deposit. This is the only effect a
  `source == target` request can have; an already-`0x02` validator cannot re-switch (it lacks
  `has_eth1_withdrawal_credential`).
- **Real consolidation** (`source_pubkey != target_pubkey`): a silent `return` on any of, in spec order — the
  `source == target` re-check; `pending_consolidations` full (== `PENDING_CONSOLIDATIONS_LIMIT`);
  `get_consolidation_churn_limit(state) <= MIN_ACTIVATION_BALANCE`; source or target pubkey unknown; source
  `!has_execution_withdrawal_credential` ∨ source `withdrawal_credentials[12:] != source_address`; **target
  `!has_compounding_withdrawal_credential` (target is not `0x02`)**; source not active; target not active; source
  `exit_epoch != FAR_FUTURE_EPOCH`; target `exit_epoch != FAR_FUTURE_EPOCH`; source too young
  (`current_epoch < source.activation_epoch + SHARD_COMMITTEE_PERIOD`); **`get_pending_balance_to_withdraw(source)
  > 0`**. Only when all pass does the CL churn-schedule the source's exit
  (`compute_consolidation_epoch_and_update_churn`) and append a `PendingConsolidation`. (Per [`06`](./06-vaults.md#key-constants):
  post-consolidation rewards above the source's effective balance sweep to the *source* WC, not the target.)

### EIP-6110 — `process_deposit_request` + `process_pending_deposits`

`process_deposit_request` does not activate a validator; it sets `deposit_requests_start_index` (if unset) and
appends a `PendingDeposit` (with `slot = state.slot`). The deposit is consumed later by `process_pending_deposits`
(which credits balance / registers the validator with `activation_epoch = FAR_FUTURE_EPOCH`; activation itself is a
further `process_registry_updates` step), finality-gated (a deposit past the finalized slot stops processing), bounded by
`MAX_PENDING_DEPOSITS_PER_EPOCH` per epoch, and churn-limited (`available_for_processing = deposit_balance_to_consume
+ get_activation_exit_churn_limit`; a deposit that does not fit the remaining churn stops processing for the epoch).
A deposit whose validator is already exiting is postponed past its `withdrawable_epoch`; one whose validator is
already withdrawn is applied to balance without consuming churn. So an EL deposit request → active validator is
**not instantaneous** — this is the window in which `PredepositGuarantee`'s WC proof lands (the WC is provable
before activation completes).

### Exit timing & churn — `initiate_validator_exit` / `compute_exit_epoch_and_update_churn`

An accepted exit is not immediate. `initiate_validator_exit` returns early if `exit_epoch != FAR_FUTURE_EPOCH`
(already exiting); otherwise it sets `exit_epoch = compute_exit_epoch_and_update_churn(effective_balance)` — the
next epoch in the balance-churn exit queue, lower-bounded by `compute_activation_exit_epoch(current_epoch)` — and
`withdrawable_epoch = exit_epoch + MIN_VALIDATOR_WITHDRAWABILITY_DELAY`. Separately, an exit can only be *initiated*
once `current_epoch >= activation_epoch + SHARD_COMMITTEE_PERIOD` (the eligibility floor enforced in
`process_withdrawal_request`, `process_voluntary_exit`, and `process_consolidation_request`); this is the same
floor `ValidatorExitDelayVerifier` reconstructs as "earliest eligible to exit" in [`08`](./08-exits.md#core-flows).
`process_voluntary_exit` **hard-asserts** `get_pending_balance_to_withdraw == 0` (a block-level assertion), where
the 7002 full-exit path treats the same condition as a silent skip. The VEDV invariant proves `exit_epoch` is
*unset* at the proven slot (no exit scheduled), not that a requested exit will be scheduled.

## Key constants

| Constant | Value | Purpose |
|---|---|---|
| `WITHDRAWAL_REQUEST` | `0x00000961Ef480Eb55e80D19ad83579A64c007002` | EIP-7002 triggerable-withdrawal predeploy (in `WithdrawalVaultEIP7002`, `TriggerableWithdrawals`). |
| `CONSOLIDATION_REQUEST_PREDEPLOY_ADDRESS` | `0x0000BBdDc7CE488642fb579F8B00f3a590007251` | EIP-7251 consolidation predeploy (in `ValidatorConsolidationRequests`). |
| `BEACON_ROOTS` | `0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02` | EIP-4788 beacon-root predeploy (in `CLProofVerifier`, `ValidatorExitDelayVerifier`). |
| SHA-256 precompile | address `0x02` | hashing for `SSZ.hashTreeRoot` and `verifyProof`. |
| `WITHDRAWAL_REQUEST_CALLDATA_LENGTH` | 56 bytes | EIP-7002 request: 48-byte pubkey + 8-byte (`WITHDRAWAL_AMOUNT_LENGTH`) big-endian `uint64` amount (`TriggerableWithdrawals`). |
| `CONSOLIDATION_REQUEST_CALLDATA_LENGTH` | 96 bytes | EIP-7251: source pubkey ‖ target pubkey (`PUBLIC_KEY_LENGTH * 2`, in `ValidatorConsolidationRequests`). |
| EIP-7002 full-exit amount | 0 | amount that signals a full exit; NO named constant in source. |
| `FAR_FUTURE_EPOCH` | `type(uint64).max` | not-exiting sentinel (VEDV invariant; `FAR_FUTURE_EPOCH` in `ValidatorExitDelayVerifier`). |
| MIN_ACTIVATION_BALANCE (CL-spec term, not an in-repo constant) | 32 ETH | 0x01 cap / min activation balance. |
| MAX_EFFECTIVE_BALANCE_ELECTRA (CL-spec term, not an in-repo constant) | 2048 ETH | 0x02 compounding cap. |
| EIP-4788 buffer | 8191 slots (`HISTORY_BUFFER_LENGTH`, prime by design) | beacon-root ring-buffer depth (~1 day); older slots use `historical_summaries`. |
| WC prefixes | `0x00` / `0x01` / `0x02` | BLS / Eth1-address / compounding. |
| GIndex packing | `(gI << 8) \| pow` | `index = bytes32 >> 8`, `pow = uint8(bytes32)`, `width = 1 << pow`. |
| `FULL_EXIT_REQUEST_AMOUNT` (CL-spec term, not an in-repo constant) | 0 | EIP-7002 full-exit sentinel `amount`; the in-repo encoder has no named constant (see the "EIP-7002 full-exit amount" row). |
| `COMPOUNDING_WITHDRAWAL_PREFIX` (CL-spec term, not an in-repo constant) | `0x02` | WC prefix `has_compounding_withdrawal_credential` matches; the 7251 switch/consolidation target prefix. |
| `PENDING_PARTIAL_WITHDRAWALS_LIMIT` (CL-spec term, not an in-repo constant) | `2**27` | partial-withdrawal queue cap; once full, only full exits process (7002). |
| `PENDING_CONSOLIDATIONS_LIMIT` (CL-spec term, not an in-repo constant) | `2**18` | consolidation queue cap; once full, consolidations skip (7251). |
| `SHARD_COMMITTEE_PERIOD` (CL-spec term; in-repo as the VEDV immutable `SHARD_COMMITTEE_PERIOD_IN_SECONDS`, denominated in seconds not epochs) | 256 epochs | min active age before an exit/consolidation can be initiated. |
| `MIN_VALIDATOR_WITHDRAWABILITY_DELAY` (CL-spec term, not an in-repo constant) | 256 epochs | `withdrawable_epoch − exit_epoch` offset applied on exit. |

## Source references

**Live source:**
- `contracts/common/lib/SSZ.sol` — `verifyProof`, `hashTreeRoot(Validator)`, `hashTreeRoot(BeaconBlockHeader)`, `toLittleEndian`; errors `InvalidProof`/`BranchHasExtraItem`/`BranchHasMissingItem`.
- `contracts/common/lib/GIndex.sol` — `pack`, `index`, `width`, `pow`, `shr`, `shl`, `concat`, `isRoot`, `unwrap`, `fls`; error `IndexOutOfRange`.
- `contracts/common/lib/BeaconTypes.sol` — `Validator`, `BeaconBlockHeader` containers.
- Consumers (boundary): `contracts/0.8.25/vaults/predeposit_guarantee/CLProofVerifier.sol` (`GI_FIRST_VALIDATOR_PREV`/`CURR`, `PIVOT_SLOT`, `GI_STATE_ROOT`, `concat`, `BEACON_ROOTS`); `contracts/0.8.25/ValidatorExitDelayVerifier.sol` (`FAR_FUTURE_EPOCH`, `GI_FIRST_HISTORICAL_SUMMARY_*`, `SLOTS_PER_HISTORICAL_ROOT`, `BEACON_ROOTS`); `contracts/0.8.9/WithdrawalVaultEIP7002.sol` + `contracts/common/lib/TriggerableWithdrawals.sol` (`WITHDRAWAL_REQUEST`); `contracts/0.8.25/vaults/ValidatorConsolidationRequests.sol` (`CONSOLIDATION_REQUEST_PREDEPLOY_ADDRESS`).

**External specs (not in repo):** EIP-7002 / EIP-7251 / EIP-4788 / EIP-6110; the Electra (Pectra) consensus-spec. **Official docs (`docs/`):** `run-on-lido/stvaults/tech-documentation/pdg.md` and `consolidation.md`.

**CL request-processing (distilled from the Electra/Pectra consensus-spec):** `process_withdrawal_request`, `process_consolidation_request` / `is_valid_switch_to_compounding_request` / `switch_to_compounding_validator` / `queue_excess_active_balance`, `process_deposit_request` / `process_pending_deposits`, `initiate_validator_exit` / `compute_exit_epoch_and_update_churn` / `compute_consolidation_epoch_and_update_churn`, `process_voluntary_exit`, `get_pending_balance_to_withdraw`, `is_eligible_for_partial_withdrawals` / `get_pending_partial_withdrawals`, `has_execution_withdrawal_credential` / `has_compounding_withdrawal_credential` / `has_eth1_withdrawal_credential`.
