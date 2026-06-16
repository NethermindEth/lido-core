---
doc: "03"
title: Oracle Accounting & Report Execution
contracts: [Accounting, Burner, OracleReportSanityChecker]
prereqs: ["00"]
see_also: ["04","06","07"]
ssot_for: [report-execution-detail, damage-bound, share-burn-timing, bad-debt-internalize, oracle-ingest]
---
# 03 — Oracle Accounting & Report Execution

> The report-execution *narrative* (where the rebase comes from, who triggers it) is the SSOT of
> [`00`](./00-architecture-overview.md#the-four-critical-flows). This doc owns the **execution detail**:
> `handleOracleReport` step ordering, the on-chain damage bound, share-burn timing, and the bad-debt
> internalize seam to [`06`](./06-vaults.md). WQ finalization continues in
> [`04`](./04-withdrawals.md#core-flows).

## Contracts

| Contract | File | Role |
|---|---|---|
| `Accounting` | `0.8.9/Accounting.sol` | Report orchestrator. Bridges `AccountingOracle` → `Lido`/`VaultHub`/`Burner`/`StakingRouter`. Bounds the rebase, finalizes WQ, internalizes vault bad debt, mints fees, emits `TokenRebased`. |
| `Burner` | `0.8.9/Burner.sol` | stETH share-burn queue. Cover vs non-cover buckets; `requestBurnShares` queues, `commitSharesToBurn` commits the aggregate and drives `Lido.burnShares` (the positive-rebase lever). |
| `OracleReportSanityChecker` | `0.8.9/sanity_checks/OracleReportSanityChecker.sol` | The sole on-chain bound on report *damage*: negative-CL-rebase (LIP-23), churn, max positive rebase, simulated-share-rate, WQ-finalization, extra-data/exit-bus counts. |
| `AccountingOracle` | `0.8.9/oracle/AccountingOracle.sol` | *Boundary (ingest).* Validates consensus data, drives `handleOracleReport`. |
| `VaultHub` | `0.8.25/vaults/VaultHub.sol` | *Boundary.* Source of `badDebtToInternalize`; bad-debt seam → [`06`](./06-vaults.md). |
| `Lido` | `0.4.24/Lido.sol` | *Boundary.* Holds share state; executes the mutations Accounting orders → [`01`](./01-core-staking.md). |

## Core flows

### 1. Report ingest seam (gating only — narrative in 00)

Gating **seam** only (committee / quorum / frame narrative in [`00`](./00-architecture-overview.md#the-four-critical-flows)); one member submits `ReportData` to `AccountingOracle.submitReportData` and the on-chain checks that matter are:

```text
HashConsensus.submitReport(refSlot, hash, consensusVersion)  // EXT: off-chain oracle daemon, quorum members
  → (>= quorum) BaseOracle.submitConsensusReport(hash, refSlot, deadline)   // stored on AccountingOracle
member/SUBMIT_DATA_ROLE → AccountingOracle.submitReportData(data, contractVersion)
  → _checkConsensusData: keccak256(data) == stored hash; refSlot is current frame; versions match
  → StakingRouter.updateExitedValidatorsCountByStakingModule(...)        // → 02
  → OracleReportSanityChecker.checkExitedValidatorsRatePerDay(...)
  → WithdrawalQueue.onOracleReport(isBunkerMode, prevTs, curTs)          // bunker flag → 04
  → Accounting.handleOracleReport(reportValues)                          // [flow 2] — all share-state changes here
  → LazyOracle.updateReportData(root, cid)                              // vault NAV root → 06
  → store extraDataHash/format/count (async per-operator exited counts follow)
External: HashConsensus quorum, off-chain daemon. `OracleDaemonConfig` is an **on-chain** `AccessControlEnumerable` key-value store (`mapping(string => bytes)`) that the off-chain daemon reads for its parameters.
```

`AccountingOracle` validates *freshness/completeness*, not magnitudes — the damage bound lives inside `handleOracleReport` ([flow 3](#core-flows)). Extra-data `itemType=1` (stuck) is deprecated and reverts `DeprecatedExtraDataType`; only `itemType=2` (exited) is live. `LegacyOracle` is no longer in the per-report path (removed in the V3 / Triggerable-Withdrawals update).

### 2. handleOracleReport — execution detail (LOAD-BEARING)

Gated `msg.sender == LidoLocator.accountingOracle` (`NotAuthorized`). The view twin `simulateOracleReport` is permissionless — the daemon calls it to pre-compute `simulatedShareRate` (empty batches) before submitting. Three phases:

```text
Accounting.handleOracleReport(ReportValues)            [only accountingOracle]
A. _snapshotPreReportState(isSimulation=false)
   → Lido.getBeaconStat / getTotalPooledEther / getTotalShares / getExternalShares / getExternalEther
   → pre.badDebtToInternalize = VaultHub.badDebtToInternalizeForLastRefSlot()   // EXT (sim twin reads live badDebtToInternalize())
B. _simulateOracleReport (view — computes deltas, no mutation)
   → WithdrawalQueue.prefinalize(batches, simulatedShareRate) → (etherToFinalizeWQ, sharesToFinalizeWQ)   // skipped if no batches/WQ paused
   → principalClBalance = pre.clBalance + (report.clValidators - pre.clValidators) * 32 ETH
   → OracleReportSanityChecker.smoothenTokenRebase(internalEther, internalShares, principal, clBalance,
       wvBalance, elBalance, sharesRequestedToBurn, etherToFinalizeWQ, sharesToFinalizeWQ)
         ⇒ (withdrawalsVaultTransfer, elRewardsVaultTransfer, sharesToBurnForWithdrawals, totalSharesToBurn)
   → _calculateProtocolFees → StakingRouter.getStakingRewardsDistribution() ⇒ (sharesToMintAsFees, feeDistribution)
   → fold sharesToMintAsFees + badDebtToInternalize into post-internal/external shares, postTotalPooledEther
C. _applyOracleReportContext (mutations, IN THIS EXACT ORDER)
   1. _sanityChecks: checkAccountingOracleReport (state-changing) + (if batches) checkSimulatedShareRate, checkWithdrawalQueueOracleReport
   2. if sharesToFinalizeWQ>0: Burner.requestBurnShares(withdrawalQueue, sharesToFinalizeWQ)   // queue, NOT commit
   3. Lido.processClStateUpdate(timestamp, preClValidators, clValidators, clBalance)
   4. if badDebtToInternalize>0: VaultHub.decreaseInternalizedBadDebt(d) + Lido.internalizeExternalBadDebt(d)  // [flow 4]
   5. if totalSharesToBurn>0: Burner.commitSharesToBurn(totalSharesToBurn)   // commits aggregate → Lido.burnShares
   6. Lido.collectRewardsAndProcessWithdrawals(...)  → pulls ELRewardsVault + WithdrawalVault into buffer, WithdrawalQueue.finalize  // EXT
   7. if sharesToMintAsFees>0: Lido.mintShares(this, fees) → _distributeFee (transferShares) → StakingRouter.reportRewardsMinted   // MINT LAST
   8. _notifyRebaseObserver → postTokenRebaseReceiver.handlePostTokenRebase(...) if registered   // EXT
   9. Lido.emitTokenRebase(...) → TokenRebased
External: AccountingOracle (caller), VaultHub, Lido, StakingRouter, Burner, WithdrawalQueue, postTokenRebaseReceiver.
```

**Why ordering matters** (full narrative: [`00`](./00-architecture-overview.md#the-four-critical-flows) flow 2). The C-phase specifics this doc owns: the source comment makes minting the "final action that changes share rate", so **fees mint last** (C.7) against the post-rebase rate; `processClStateUpdate` (C.3) runs before the burns so beacon state is set before the rate moves; the WQ burn is **queued** in C.2 (`requestBurnShares`) but **committed as the aggregate** in C.5 (`commitSharesToBurn(totalSharesToBurn)` = cover/non-cover **plus** WQ shares); and `ReportValues` carries **no vault NAV/fee fields** — V3 vault reporting is the separate `LazyOracle.updateReportData` path ([`06`](./06-vaults.md#core-flows)), the only vault touch here being bad-debt internalize (C.4).

### 3. Sanity bound — checkAccountingOracleReport (LOAD-BEARING)

`OracleReportSanityChecker` is the **one on-chain bound on report damage**. `checkAccountingOracleReport` is state-changing (appends `ReportData` history), gated `msg.sender == ACCOUNTING_ADDRESS` (`CalledNotFromAccounting`). Six sub-checks; the load-bearing one is the negative-CL-rebase backstop:

```text
checkAccountingOracleReport(timeElapsed, preCLBalance, postCLBalance, wvBalance, elBalance, sharesReqToBurn, preCLVals, postCLVals)
  1. wvBalance      <= withdrawalVault.balance         (else IncorrectWithdrawalsVaultBalance)   // freshness, not magnitude
  2. elBalance      <= elRewardsVault.balance          (else IncorrectELRewardsVaultBalance)
  3. sharesReqToBurn <= Burner cover+nonCover requested (else IncorrectSharesRequestedToBurn)
  4. _checkCLBalanceDecrease  → LIP-23 negative-rebase backstop  (see below)
  5. _checkAnnualBalancesIncrease  (annualBalanceIncreaseBPLimit, bps)
  6. if postCLVals>preCLVals: _checkAppearedValidatorsChurnLimit (appearedValidatorsPerDayLimit)
```

**Negative-CL-rebase backstop (LIP-23).** When `preCLBalance > postCLBalance + wvBalance`, the drop is
recorded and the protocol sums all negative rebases newer than `reportTimestamp − 18 days`. The allowed
envelope is built from validator counts at two LIP-23 windows:

```text
maxAllowedNegativeSum =
    initialSlashingAmountPWei  * 1e15 * (postCLValidators − exitedValidatorsAt(ts − 18 days))
  + inactivityPenaltiesAmountPWei * 1e15 * (postCLValidators − exitedValidatorsAt(ts − 54 days))
if negativeCLRebaseSum <= maxAllowedNegativeSum: accept (NegativeCLRebaseAccepted)
elif secondOpinionOracle == 0: revert IncorrectCLBalanceDecrease
else: _askSecondOpinion(...)
```

If the drop exceeds the envelope and a **Second Opinion Oracle** is configured, `_askSecondOpinion` calls `secondOpinionOracle.getReport(refSlot)` and requires **all** of: the call succeeded (else `NegativeRebaseFailedSecondOpinionReportIsNotReady`); the oracle's CL balance is **not below** the reported `postCLBalance`; within `clBalanceOraclesErrorUpperBPLimit` of it (upper-margin — the second opinion may read slightly higher); and its withdrawal-vault balance **exactly equals** the reported `wvBalance` (else `NegativeRebaseFailedWithdrawalVaultBalanceMismatch`). Any failure reverts the report. This is the primary defense against a malicious daemon under-reporting CL balance to fabricate a loss.

**`smoothenTokenRebase` (clamp, not revert).** Caps EL+CL upside per report at `maxPositiveTokenRebase` (1e9 precision; defeats oracle-sandwiching MEV). A report over the cap is *clamped* — surplus ETH stays in the vaults, surplus shares stay unburnt for the next report. Also returns `sharesFromWQToBurn` (= `sharesToBurn − simulatedSharesToBurn`), the WQ-only slice used to verify `simulatedShareRate`. Inputs are **internal** ether/shares (`pre.totalPooledEther − externalEther`, `pre.totalShares − externalShares`) — the rate excludes external vault shares.

**`checkSimulatedShareRate`** (only with batches): virtually returns the WQ-locked ether/shares to post-state and requires the submitted `simulatedShareRate` within `simulatedShareRateDeviationBPLimit` of recomputed `postInternalEther * 1e27 / postInternalShares`. **`checkWithdrawalQueueOracleReport`** requires the last finalizable request at least `requestTimestampMargin` old, blocking finalization of fresh requests at a stale rate (consumed in [`04`](./04-withdrawals.md#internal-mechanics)).

### 4. Bad-debt internalize seam (LOAD-BEARING) → 06

When a stVault is underwater, its loss is socialized onto internal stETH holders. Accounting reads the value
at snapshot, then settles it atomically in C.4:

```text
A: pre.badDebtToInternalize = VaultHub.badDebtToInternalizeForLastRefSlot()   // EXT (sim: live badDebtToInternalize())
C.4 (if > 0):
  → VaultHub.decreaseInternalizedBadDebt(badDebt)        // EXT: VaultHub debits its tracked obligation → 06
  → Lido.internalizeExternalBadDebt(badDebt)             // EXT: moves badDebt shares external → internal
```

In `_simulateOracleReport` the bad debt is folded as `postInternalShares += badDebt` and `postExternalShares = externalShares − badDebt` ("can't underflow by design"). Net: external shares shrink, internal shares grow by the same count, so internal-holder share rate **drops** — the vault loss is borne by stakers. The two calls must stay paired (VaultHub debit + Lido move) or the books desync — the socialize-vs-internalize double-settle surface. How `VaultHub` accrues bad debt: [`06`](./06-vaults.md#core-flows). **Sim/exec divergence:** the live `badDebtToInternalize()` (permissionless simulate twin) and the snapshotted `…ForLastRefSlot()` (on-chain) can differ — the daemon must simulate against the current ref-slot view.

### 5. Burner — share-burn queue (RESTORED)

Burns *decrease* `totalShares` to effect a positive rebase. Requests sit pending (held as stETH on the Burner)
until the next report commits them. Two role-gated entry families plus the cover/non-cover split:

```text
// queue from `from` (pulls via Lido.transferSharesFrom)  — REQUEST_BURN_SHARES_ROLE (= Accounting + CSM_ACCOUNTING)
caller → requestBurnShares(from, shares)            → nonCoverSharesBurnRequested += shares
caller → requestBurnSharesForCover(from, shares)    → coverSharesBurnRequested   += shares
// self-burn (pulls from msg.sender)                    — REQUEST_BURN_MY_STETH_ROLE
self → requestBurnMyStETHForCover(stETH)            → cover bucket
self → requestBurnMyStETH(stETH) / requestBurnMyShares(shares)  → non-cover bucket   (requestBurnMyStETH deprecated: dust)
// commit (during a report)                             — msg.sender == LidoLocator.accounting()
Accounting → commitSharesToBurn(total)              → drains cover-first then non-cover (Math.min per bucket)
                                                      → Lido.burnShares(total); assert(burnedNow == total)
External: Lido.transferSharesFrom (pull), Lido.burnShares (commit).
```

`commitSharesToBurn(total)` reverts `BurnAmountExceedsActual` if `total > cover+nonCover requested`; drains cover-first, updates lifetime `totalCover/NonCoverSharesBurnt`, then `Lido.burnShares(total)` and asserts the per-bucket sum equals `total`. **Cover vs non-cover** is informational only (integrators split a rebase into rewards vs insurance via `getCoverSharesBurnt`/`getNonCoverSharesBurnt`); supply impact identical. `requestBurnShares` holders are **`ACCOUNTING` + `CSM_ACCOUNTING` only** (see [`07`](./07-governance-permissions.md#contracts)).

### 6. Burner — excess-stETH recovery and migrate (RESTORED)

stETH sent to the Burner *outside* the request path is not auto-burnt — it sits as `getExcessStETH()` (= `sharesOf(Burner) − coverRequested − nonCoverRequested`, via `_getExcessStETHShares`). Recovery is **permissionless by design** and hard-wired to treasury, so accidental sends are never burned irrecoverably:

```text
anyone → recoverExcessStETH()          → Lido.transfer(LOCATOR.treasury(), excessStETH)        [permissionless]
anyone → recoverERC20(token, amount)   → token.safeTransfer(treasury)   (revert StETHRecoveryWrongFunc if token==LIDO)
anyone → recoverERC721(token, id)      → token.transferFrom(this, treasury)  (same stETH guard)
receive() → revert DirectETHTransfer   // Burner rejects raw ETH
```

Recovery cannot touch shares marked for burning (requested buckets subtracted) and only sends to `LOCATOR.treasury()`. **V3 migrate:** `migrate(oldBurner)` is `msg.sender == LIDO` only (`OnlyLidoCanMigrate`), gated on `isMigrationAllowed` (set at `initialize`), flips that flag false so it runs once; copies `totalCover/NonCoverSharesBurnt` and the requested buckets from the old Burner. `initialize(admin, isMigrationAllowed)` is one-time, grants `DEFAULT_ADMIN_ROLE`.

## Internal mechanics

- **Share rate is internal-only** — every rate computation (`smoothenTokenRebase`, fees, `checkSimulatedShareRate`) uses `totalPooledEther − externalEther` over `totalShares − externalShares`; the `1e27` `SHARE_RATE_PRECISION_E27` is *computation* precision, not token scaling (share-math SSOT: [`01`](./01-core-staking.md#internal-mechanics)).
- **Fees only when profitable (LIP-12).** `_calculateTotalProtocolFeeShares` mints only when `clBalance + withdrawalsVaultTransfer > principalClBalance`; then `feeEther = totalRewards * totalFee / precisionPoints`, `sharesToMintAsFees = feeEther * internalSharesBeforeFees / (postInternalEther − feeEther)` — derived so minted shares exactly compensate the fee at the post-rebase rate. `_distributeFee` `transferShares` to each module recipient (residual to treasury).
- **Pre-mutation guards in `_sanityChecks`.** `report.timestamp < block.timestamp` (`IncorrectReportTimestamp`); `preClValidators <= report.clValidators <= depositedValidators` (`IncorrectReportValidators`); `postInternalShares != 0` (`InternalSharesCantBeZero`).
- **LimitsList packed** into `LimitsListPacked` (uint16/uint32/uint64); read via `unpack()` per call. History (`ReportData[]`) appended on every `checkAccountingOracleReport`, walked backward by timestamp for the 18/54-day LIP-23 sums.
- **Ceiling, not equality, on EL/WV/burn inputs.** Reported `wvBalance`/`elBalance`/`sharesRequestedToBurn` must be `<=` live on-chain values — the report can under-claim but never over-claim.

## External interactions

```text
Accounting
  ← AccountingOracle.submitReportData → handleOracleReport            (gated msg.sender == accountingOracle)
  → VaultHub.badDebtToInternalizeForLastRefSlot                       (pre-state read)
  → OracleReportSanityChecker.smoothenTokenRebase / checkAccountingOracleReport / checkSimulatedShareRate / checkWithdrawalQueueOracleReport
  → StakingRouter.getStakingRewardsDistribution / reportRewardsMinted
  → Burner.requestBurnShares (WQ shares) / commitSharesToBurn (aggregate)
  → Lido.processClStateUpdate
  → VaultHub.decreaseInternalizedBadDebt + Lido.internalizeExternalBadDebt   [bad-debt internalize → 06]
  → Lido.collectRewardsAndProcessWithdrawals (pulls WithdrawalVault + ELRewardsVault, finalizes WithdrawalQueue → 04)
  → Lido.mintShares → _distributeFee → emitTokenRebase (TokenRebased)
  → postTokenRebaseReceiver.handlePostTokenRebase (if registered)

Burner
  ← REQUEST_BURN_SHARES_ROLE (Accounting, CSM_ACCOUNTING)            → requestBurnShares[ForCover]
  ← REQUEST_BURN_MY_STETH_ROLE                                       → requestBurnMy*
  ← msg.sender == accounting                                         → commitSharesToBurn
  ← msg.sender == LIDO                                               → migrate (one-time)
  ← anyone                                                           → recoverExcessStETH / recoverERC20 / recoverERC721 → treasury
  → Lido.transferSharesFrom / burnShares / transfer

OracleReportSanityChecker
  ← Accounting (checkAccountingOracleReport gated; smoothen/checkSimulated views)
  ← AccountingOracle (checkExitedValidatorsRatePerDay, checkExtraData*)
  ← ValidatorsExitBusOracle (checkExitBusOracleReport → 08)
  → secondOpinionOracle.getReport (LIP-23 backstop)
  → StakingRouter.getStakingModule* / Burner.getSharesRequestedToBurn / WithdrawalQueue.getWithdrawalStatus (reads)
```

## Key constants

| Constant | Value | Purpose |
|---|---|---|
| `DEPOSIT_SIZE` (Accounting) | `32 ether` | principalClBalance per-validator unit (pre-maxEB) |
| `SHARE_RATE_PRECISION_E27` | `1e27` | share-rate / simulatedShareRate computation precision |
| `ONE_PWEI` | `1e15` | LIP-23 slashing/penalty unit (pWei) |
| `MAX_BASIS_POINTS` | `10000` | 100% in bps for deviation/margin checks |
| `maxPositiveTokenRebase` | 1e9 precision (1e6 = 0.1%) | per-report positive-rebase clamp |
| LIP-23 windows | 18 days / 54 days | negative-rebase sum window / inactivity-penalty validator window |
| `initialSlashingAmountPWei` | LimitsList (pWei) | expected slash per at-risk validator |
| `inactivityPenaltiesAmountPWei` | LimitsList (pWei) | expected inactivity penalty per validator |
| `clBalanceOraclesErrorUpperBPLimit` | LimitsList (bps) | second-opinion CL-balance upper margin |
| `EXTRA_DATA_TYPE_EXITED_VALIDATORS` | 2 | only live extra-data item type (type 1 stuck = deprecated) |
| `REQUEST_BURN_SHARES_ROLE` | keccak256 | Accounting + CSM_ACCOUNTING only |
| `REQUEST_BURN_MY_STETH_ROLE` | keccak256 | self-burn entry points |

## Source references

**Live source** (symbols cited above resolve here):
- `contracts/0.8.9/Accounting.sol` — `handleOracleReport` (gated `accountingOracle`), `simulateOracleReport` (view, permissionless), `_snapshotPreReportState`/`_simulateOracleReport`/`_applyOracleReportContext` (A/B/C), `_calculateTotalProtocolFeeShares`, `_sanityChecks`.
- `contracts/0.8.9/Burner.sol` — `REQUEST_BURN_SHARES_ROLE`/`REQUEST_BURN_MY_STETH_ROLE`, `requestBurnShares`/`requestBurnSharesForCover`/`requestBurnMy*`, `commitSharesToBurn` (accounting-gated), `getExcessStETH`/`recoverExcessStETH`/`recoverERC20`/`recoverERC721`, `migrate` (Lido-only), `initialize`.
- `contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol` — `LimitsList`, `smoothenTokenRebase`, `checkAccountingOracleReport` (gated `CalledNotFromAccounting`), `_checkCLBalanceDecrease`/`_askSecondOpinion` (LIP-23), `checkSimulatedShareRate`/`checkWithdrawalQueueOracleReport`/`checkExitedValidatorsRatePerDay`/`checkExtraData*`/`checkExitBusOracleReport`, `setOracleReportLimits`.

**Official docs (`context/docs/...`):** `contracts/burner.md`, `contracts/oracle-report-sanity-checker.md`, `contracts/accounting-oracle.md`, `contracts/lido.md`.
