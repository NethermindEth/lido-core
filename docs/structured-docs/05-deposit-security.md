---
doc: "05"
title: Deposit Security (boundary seam)
contracts: [DepositSecurityModule, BeaconChainDepositor]
prereqs: ["01"]
see_also: ["02"]
ssot_for: [deposit-gate, guardian-quorum]
---
# 05 — Deposit Security (boundary seam)

> **Deposit-gate seam.** `DepositSecurityModule` (DSM) sits between `Lido.deposit` (→ [`01`](./01-core-staking.md))
> and the on-chain deposit. It holds no shares/funds and is outside the rebase invariant; the dependency is
> reversed — `Lido.deposit` requires `msg.sender == locator.depositSecurityModule()`. This doc owns 0 of the 39
> in-scope contracts; it covers the seam to stress-test from the in-scope side: guardian ECDSA quorum, the
> check-order, the allow-by-quorum vs deny-by-one asymmetry, the deposit-root front-run check, and the unvet
> path into `StakingRouter` (→ [`02`](./02-staking-router-modules.md)).

## Contracts

| Contract | File | Role |
|---|---|---|
| `DepositSecurityModule` | `0.8.9/DepositSecurityModule.sol` | Guardian committee; quorum-gated deposit caller; pause-on-attest; unvet relay. Boundary (not in the 39). |
| `BeaconChainDepositor` | `0.8.9/BeaconChainDepositor.sol` | Mixin inherited by `StakingRouter`; pushes 32-ETH batches to `IDepositContract`. Boundary; covered in [`02`](./02-staking-router-modules.md). |

DSM uses an `owner`/`onlyOwner` model (not OZ AccessControl); the owner is Lido governance (Agent, governed by DG). Its address is the only caller `Lido.deposit` accepts. Three EIP-712-style prefixes (`ATTEST_MESSAGE_PREFIX`, `PAUSE_MESSAGE_PREFIX`, `UNVET_MESSAGE_PREFIX`), each = `keccak256(tag, block.chainid, address(this))`, are immutable (set in constructor), so a signature for one DSM cannot replay across deployments or chains.

## Core flows

### 1. Deposit gate (`depositBufferedEther`)

Mitigates the deposit front-running vuln (LIP-5): an attacker who controls a validator key pre-funds it with their own withdrawal credentials right before the protocol's 32 ETH lands, redirecting it. DSM only releases buffered ETH after a guardian `quorum` has signed the current `(blockNumber, blockHash, depositRoot, stakingModuleId, nonce)` tuple. The `depositRoot` check is the heart of the mitigation: any intervening malicious predeposit changes `get_deposit_root()`, so the signed root no longer matches and the call reverts — the attacker must outrun block production or get a guardian to sign a stale root (collusion).

```text
off-chain bot: poll deposit_root + module nonce, collect >= quorum guardian ECDSA sigs
  -> DSM.depositBufferedEther(blockNumber, blockHash, depositRoot, stakingModuleId, nonce,
                              depositCalldata, sortedGuardianSignatures[])
     // checks in EXACTLY this order (early checks = cheapest / most-likely-stale first):
     1 depositRoot == IDepositContract.get_deposit_root()          // EXT: beacon deposit contract; else DepositRootChanged
     2 StakingRouter.getStakingModuleNonce(id) == nonce            // EXT: SR; else ModuleNonceChanged
     3 quorum != 0 && sigs.length >= quorum                        // else DepositNoQuorum
     4 StakingRouter.getStakingModuleIsActive(id)                  // EXT: SR; else DepositInactiveModule
     5 block.number - max(lastDepositBlock, SR.lastDepositBlock(id)) >= SR.minDepositBlockDistance(id)
                                                                   // else DepositTooFrequent
     6 blockHash != 0 && blockhash(blockNumber) == blockHash       // else DepositUnexpectedBlockHash
     7 !isDepositsPaused                                           // else DepositsArePaused
     8 _verifyAttestSignatures: each sig recovers to a guardian, strictly ASCENDING by address
                                                                   // else InvalidSignature / SignaturesNotSorted
  -> maxDepositsPerBlock = StakingRouter.getStakingModuleMaxDepositsPerBlock(id)   // EXT: SR; cap read live, not stored
  -> Lido.deposit(maxDepositsPerBlock, id, depositCalldata)        // EXT: Lido (caps by buffered ether) -> StakingRouter.deposit
  -> _setLastDepositBlock(block.number)                           // resets the GLOBAL rate-limit window
External: beacon deposit contract (root); StakingRouter (nonce/active/cap/distance); Lido; off-chain guardian quorum.
```

Any failed check reverts the whole call — no partial state. The strictly-ascending signer ordering both rejects duplicate guardian signatures (one guardian can't fill quorum twice) and bounds the loop. `_setLastDepositBlock` updates DSM's own `lastDepositBlock`, and step 5 takes the `max` of it and the per-module last-deposit block — so a deposit to **any** module resets the cooldown for **every** module. That is intentional: it denies a colluding guardian subset the chance to race parallel deposits across modules in one block before an honest guardian can `pauseDeposits`.

### 2. Pause (`pauseDeposits`) — deny-by-one

```text
guardian observes a malicious predeposit (single guardian, NO quorum)
  -> DSM.pauseDeposits(blockNumber, sig)        // guardian's own tx, or anyone relays a guardian's sig
     1 if isDepositsPaused: return              // idempotent (no revert) — all guardians may call
     2 if msg.sender not a guardian: recover sig over (PAUSE_MESSAGE_PREFIX, blockNumber) -> must be guardian
                                                // else InvalidSignature
     3 block.number - blockNumber <= pauseIntentValidityPeriodBlocks   // else PauseIntentExpired
  -> isDepositsPaused = true; emit DepositsPaused
External: off-chain guardian committee; signed pause message relayable by any address.
```

The defining property: a **single** guardian pauses (no quorum), so N−1 colluding guardians cannot lock the honest one out of blocking a bad deposit. The asymmetry — `quorum` signatures to ALLOW, one to DENY — is the primary defense against guardian collusion. There is no on-chain way to challenge a pause; a false pause costs only reputation. Resume is owner-only (see Internal mechanics — errata #8: the function is `unpauseDeposits`, not `resumeDeposits`).

### 3. Unvet (`unvetSigningKeys`) — deny-by-one, additive to pause

Like pause, the gate is a **single** guardian (`msg.sender` is a guardian, or one valid guardian signature) — **no quorum**. Unvetting reduces an operator's vetted-key cap below a flagged key's index, so that specific key can never be deposited even after deposits resume. It is additive to pause and uses a single compact `Signature{r, vs}` (EIP-2098), not a sorted array; `maxOperatorsPerUnvetting` bounds how many operators one call touches.

```text
guardian flags compromised keys for one or more operators
  -> DSM.unvetSigningKeys(blockNumber, blockHash, stakingModuleId, nonce,
                          nodeOperatorIds, vettedSigningKeysCounts, sig)
     1 StakingRouter.getStakingModuleNonce(id) == nonce            // EXT: SR; else ModuleNonceChanged
     2 nodeOperatorIds packed 8 bytes/id; vettedSigningKeysCounts packed 16 bytes/count;
       counts == ids count; ids count <= maxOperatorsPerUnvetting  // else UnvetPayloadInvalid
     3 msg.sender is guardian, else recover sig -> must be guardian // else InvalidSignature
     4 blockHash != 0 && blockhash(blockNumber) == blockHash        // else UnvetUnexpectedBlockHash
  -> StakingRouter.decreaseStakingModuleVettedKeysCountByNodeOperator(id, ids, counts)
                                                                    // EXT: SR; needs STAKING_MODULE_UNVETTING_ROLE
External: StakingRouter (nonce read + vetted-cap decrease); off-chain guardian committee.
```

The `StakingRouter` target only **decreases** the cap — there is no path here to raise trust in an operator. The nonce check (step 1) is the staleness guard: a successful unvet bumps the module nonce, which in turn invalidates any in-flight ATTEST signatures for that module (their `nonce` no longer matches in flow 1 step 2).

## Internal mechanics

- **Storage:** `isDepositsPaused` (bool), `lastDepositBlock`, `pauseIntentValidityPeriodBlocks`, `maxOperatorsPerUnvetting`, `owner`, `quorum`, `guardians[]`, `guardianIndicesOneBased` (one-based map; index 0 = absent, getter returns int256 `-1`). DSM stores no per-block deposit count — the cap is read live from `StakingRouter` at deposit time.
- **`blockhash` window:** `blockhash(blockNumber)` returns 0 for future or >256-block-old blocks, so both forward-dated and stale ATTEST/UNVET intents revert on the `blockHash` check.
- **Owner knobs (`onlyOwner`):** `setOwner` (reverts `ZeroAddress` — owner cannot be renounced to 0), `setPauseIntentValidityPeriodBlocks` / `setMaxOperatorsPerUnvetting` (revert `ZeroParameter` on 0), `setGuardianQuorum`, `addGuardian` / `addGuardians` / `removeGuardian` (each re-sets quorum; remove is swap-pop), and `unpauseDeposits`. Errata #8: the resume function is `unpauseDeposits()` — it reverts `DepositsNotPaused` if not paused; `resumeDeposits()` does not exist.
- **Quorum edge cases:** `setGuardianQuorum` may set `quorum` ABOVE `guardians.length` (explicitly permitted) — a soft kill-switch that blocks deposits without touching the pause flag. It may also shrink quorum, unilaterally loosening the gate. `quorum == 0` always fails flow 1 step 3.

## External interactions

```text
DepositSecurityModule (boundary)
  <- depositor bot   : depositBufferedEther / unvetSigningKeys / pauseDeposits
  <- any guardian    : pauseDeposits, unvetSigningKeys (via msg.sender)
  <- owner (gov)     : setOwner, setGuardianQuorum, add/removeGuardian(s),
                       setPauseIntentValidityPeriodBlocks, setMaxOperatorsPerUnvetting, unpauseDeposits
  -> IDepositContract.get_deposit_root()                                   // ATTEST verification
  -> StakingRouter.getStakingModuleNonce / .getStakingModuleIsActive / .hasStakingModule /
     .getStakingModuleLastDepositBlock / .getStakingModuleMinDepositBlockDistance /
     .getStakingModuleMaxDepositsPerBlock                                  // read-only gate checks + cap
  -> StakingRouter.decreaseStakingModuleVettedKeysCountByNodeOperator      // UNVET; STAKING_MODULE_UNVETTING_ROLE
  -> Lido.deposit / Lido.canDeposit                                        // deposit kickoff / view

BeaconChainDepositor (mixed into StakingRouter — see 02)
  -> IDepositContract.deposit{value: 32 ether}                             // the actual 32-ETH push
```

DSM has no role on `Lido` and no role on `StakingRouter` except `STAKING_MODULE_UNVETTING_ROLE`. The link to `Lido.deposit` is by **address equality** against `LidoLocator.depositSecurityModule()`; changing that locator entry replaces the security committee wholesale. `BeaconChainDepositor._makeBeaconChainDeposits32ETH` loops `keysCount` 32-ETH `deposit` calls, computing `deposit_data_root` in-place (`_computeDepositDataRoot`); it does NOT verify BLS signatures (nor does the deposit contract — only the data root + 32-ETH amount are checked), so a malformed signature still succeeds on-chain but yields a non-activating validator. Module withdrawal credentials come from `StakingRouter.getWithdrawalCredentials()` (0x01 = `WithdrawalVault` for Curated/CSM; stVaults deposit via PDG with per-vault 0x02 credentials, → [`06`](./06-vaults.md)).

## Key constants

| Constant | Value | Purpose |
|---|---|---|
| `VERSION` | 3 | DSM contract version |
| `DEPOSIT_SIZE` | 32 ether | `BeaconChainDepositor` per-validator deposit |
| `DEPOSIT_SIZE_IN_GWEI_LE64` | `0x0040597307000000` | 32 ETH in gwei, little-endian uint64, for `deposit_data_root` |
| `PUBLIC_KEY_LENGTH` | 48 | BLS pubkey bytes (per-key slice) |
| `SIGNATURE_LENGTH` | 96 | BLS signature bytes (per-key slice) |
| `quorum` | configurable | ATTEST signatures to ALLOW a deposit (may exceed `guardians.length`) |
| `pauseIntentValidityPeriodBlocks` | configurable, != 0 | PAUSE intent freshness window |
| `maxOperatorsPerUnvetting` | configurable, != 0 | upper bound on operators per `unvetSigningKeys` call |
| unvet packing | 8 bytes/id, 16 bytes/count | `nodeOperatorIds` / `vettedSigningKeysCounts` calldata layout |

## Source references

**Live source:**
- `0.8.9/DepositSecurityModule.sol` — owner/guardian model; `ATTEST`/`PAUSE`/`UNVET` prefixes; `depositBufferedEther` (check order) + `_verifyAttestSignatures`; `pauseDeposits` (idempotent return); `unpauseDeposits` (reverts `DepositsNotPaused`); `unvetSigningKeys`; `_isMinDepositDistancePassed`; guardian/quorum knobs.
- `0.8.9/BeaconChainDepositor.sol` — `_makeBeaconChainDeposits32ETH`, `_computeDepositDataRoot`, `DEPOSIT_SIZE`, `DEPOSIT_SIZE_IN_GWEI_LE64` (reviewed via `StakingRouter`; see [`02`](./02-staking-router-modules.md)).
- `0.8.9/StakingRouter.sol` — `STAKING_MODULE_UNVETTING_ROLE` and `decreaseStakingModuleVettedKeysCountByNodeOperator` (UNVET target).

**Official docs (context/docs/...):**
- `contracts/deposit-security-module.md` — guardian quorum (4/6), message types, view/admin methods.
- `guides/deposit-security-manual.md` — front-running vuln (LIP-5), Deposit Security Committee, threat model.
- `contracts/staking-router.md` — unvet target and deposit dispatch.
