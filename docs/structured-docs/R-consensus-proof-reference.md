---
doc: "R"
title: Consensus & Proof Reference
contracts: [SSZ, GIndex]
prereqs: []
see_also: ["04","06","08"]
ssot_for: [ssz-gindex, proof-model, eip-7002, eip-7251, eip-4788, withdrawal-credentials]
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
the `(pubkey, withdrawalCredentials)` parent subtree node via `BLS12_381.sha256Pair(pubkeyRoot, wc)` and binds that
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
                 -> revert RootNotFound if empty (slot outside the 8192-slot ring buffer)

VEDV (ValidatorExitDelayVerifier):
  1. require root == header.hashTreeRoot()                       // _verifyBeaconBlockRoot: equality, NOT verifyProof
  2. SSZ.verifyProof(validatorProof, header.stateRoot,
                     validator.hashTreeRoot(),                    // leaf has exitEpoch hard-coded to FAR_FUTURE_EPOCH
                     _getValidatorGI(validatorIndex, header.slot))

PDG (CLProofVerifier):
  1. leaf = sha256Pair(pubkeyRoot(pubkey), withdrawalCredentials) // parent(pubkey, wc) subtree node
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
(`_verifyBeaconBlockRoot`), so it neither imports `CLProofVerifier` nor calls `_verifySlot`. For slots **older than the 8192-slot buffer**, step 0 cannot resolve the timestamp; the verifier
instead proves the old block root through the beacon state's `historical_summaries` accumulator (Capella+): with
`targetSlot` and `recentSlot`, compute `summaryIndex = (targetSlot − CAPELLA_SLOT) / SLOTS_PER_HISTORICAL_ROOT` and
`rootIndex = targetSlot % SLOTS_PER_HISTORICAL_ROOT`, then `verifyProof` the old header against the recent block's
`stateRoot` using `_getHistoricalBlockRootGI` (`GI_FIRST_HISTORICAL_SUMMARY_*.shr(summaryIndex)` then
`.concat(GI_FIRST_BLOCK_ROOT_IN_SUMMARY_*).shr(rootIndex)`, each PREV/CURR-selected by slot). The recent anchor (a
still-buffered block) is the EIP-4788 root; everything else hangs off it by SSZ branches.

## Distilled external specs

Lido-relevant facts only (distilled, not transcribed). Predeploy addresses verified against source constants below.

- **EIP-7002 (triggerable withdrawals/exits).** Request = **56 bytes** (`WITHDRAWAL_REQUEST_CALLDATA_LENGTH`) =
  48-byte pubkey ++ big-endian `uint64` amount (`WITHDRAWAL_AMOUNT_LENGTH = 8`); `amount = 0` ⇒ full exit (no
  named constant in source — the source comment is "withdrawal amount = 0"). Submitted to predeploy `…007002`
  (`WITHDRAWAL_REQUEST` in both `WithdrawalVaultEIP7002` and `common/lib/TriggerableWithdrawals`). Fee is dynamic
  (EIP-1559-style on a queue-excess counter): fee getter is `predeploy.staticcall("")` returning a `uint256`;
  the wrapper sends `call{value: fee}(request)` and refunds `msg.value − totalFee`. The system contract uses the
  caller (`msg.sender`) as the withdrawal-request source, so the calling contract must hold the 0x01-source credential.
  *(One-line refs — never transcribed: in-state queue layout, dequeue/excess-update/count-reset helpers, synthetic
  deployment blob, 30M system-call gas, EIP-7685 wrapping.)*
- **EIP-7251 (consolidation).** Predeploy `…007251` (`CONSOLIDATION_REQUEST_PREDEPLOY_ADDRESS`). Calldata = **96
  bytes** = source pubkey ‖ target pubkey (`2 × 48`). Merges a source validator into a `0x02` (compounding) target;
  switching `0x01 → 0x02` raises a validator's effective-balance cap. Same staticcall("") fee/refund shape as 7002.
  In-repo encoder is `ValidatorConsolidationRequests` (Vault CLI; holds no funds, mutates no state).
- **EIP-4788 (beacon roots).** `BEACON_ROOTS` predeploy `0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02`
  (identical constant in `CLProofVerifier` and `ValidatorExitDelayVerifier`). Timestamp-keyed **8192-slot ring
  buffer** of beacon block roots; `staticcall(abi.encode(timestamp))` returns the root or empty. Slots older than
  the buffer use the `historical_summaries` fallback (flow 4).
- **Withdrawal-credential types.** `0x00` BLS (legacy, not used for new Lido validators); `0x01` Eth1-address
  (`0x01 ‖ 11 zero bytes ‖ 20-byte addr`, Core Pool → Lido `WithdrawalVault`, CL-spec cap MIN_ACTIVATION_BALANCE =
  32 ETH); `0x02` compounding (Electra, V3 `StakingVault` = `0x02 ‖ 12 zero bytes ‖ address(this)`, CL-spec cap
  MAX_EFFECTIVE_BALANCE_ELECTRA = 2048 ETH, required for 7251). These caps are CL-consensus-spec terms, not in-repo
  constants (only `MIN_ACTIVATION_BALANCE` appears, in a `Dashboard` doc-comment). **Invariant:** a validator never
  requested-and-initiated to exit has `exitEpoch == FAR_FUTURE_EPOCH` (`type(uint64).max`) — the property VEDV
  proves (the leaf is reconstructed with `exitEpoch` hard-coded to `FAR_FUTURE_EPOCH`). On exit, `withdrawableEpoch
  = exitEpoch + MIN_VALIDATOR_WITHDRAWABILITY_DELAY` (a fixed CL-spec offset).
- **EIP-6110 (deposit requests).** EL deposit-request → validator activation is **not instantaneous**: it enters
  the activation queue and is churn-limited (`get_activation_exit_churn_limit`), so PDG's WC proof can predate
  activation.

*(SUMMARIZE-not-copy, one-line each: Pectra committee/attestation EIP-7549, sync-committee, blob EIP-7691,
proposer-index, `process_slashings`, full `BeaconState`/`BeaconBlockBody` dumps, engine APIs,
`process_pending_deposits`/churn detail, sweep machinery — all out of in-repo scope.)*

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
| EIP-4788 buffer | 8192 slots | beacon-root ring-buffer depth; older slots use `historical_summaries`. |
| WC prefixes | `0x00` / `0x01` / `0x02` | BLS / Eth1-address / compounding. |
| GIndex packing | `(gI << 8) \| pow` | `index = bytes32 >> 8`, `pow = uint8(bytes32)`, `width = 1 << pow`. |

## Source references

**Live source:**
- `contracts/common/lib/SSZ.sol` — `verifyProof`, `hashTreeRoot(Validator)`, `hashTreeRoot(BeaconBlockHeader)`, `toLittleEndian`; errors `InvalidProof`/`BranchHasExtraItem`/`BranchHasMissingItem`.
- `contracts/common/lib/GIndex.sol` — `pack`, `index`, `width`, `pow`, `shr`, `shl`, `concat`, `isRoot`, `unwrap`, `fls`; error `IndexOutOfRange`.
- `contracts/common/lib/BeaconTypes.sol` — `Validator`, `BeaconBlockHeader` containers.
- Consumers (boundary): `contracts/0.8.25/vaults/predeposit_guarantee/CLProofVerifier.sol` (`GI_FIRST_VALIDATOR_PREV`/`CURR`, `PIVOT_SLOT`, `GI_STATE_ROOT`, `concat`, `BEACON_ROOTS`); `contracts/0.8.25/ValidatorExitDelayVerifier.sol` (`FAR_FUTURE_EPOCH`, `GI_FIRST_HISTORICAL_SUMMARY_*`, `SLOTS_PER_HISTORICAL_ROOT`, `BEACON_ROOTS`); `contracts/0.8.9/WithdrawalVaultEIP7002.sol` + `contracts/common/lib/TriggerableWithdrawals.sol` (`WITHDRAWAL_REQUEST`); `contracts/0.8.25/vaults/ValidatorConsolidationRequests.sol` (`CONSOLIDATION_REQUEST_PREDEPLOY_ADDRESS`).

**Official docs (context/docs/...):** EIP-7002 / EIP-7251 / EIP-4788 / EIP-6110 specs; the Electra (Pectra) consensus-spec; `run-on-lido/stvaults/tech-documentation/pdg.md` and `consolidation.md`.
