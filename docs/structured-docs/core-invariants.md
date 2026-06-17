---
doc: "core-invariants"
title: Core Invariants
contracts: []
prereqs: ["AGENTS-1"]
see_also: ["01", "02", "03", "04"]
ssot_for: []
---
# Invariants — Supplementary Properties (Core)

> A curated set of properties worth verifying in the Core protocol, expressed as machine-readable invariants. This file is **supplementary context, not the full audit-target set** — the agent should treat these as known properties to keep in mind and should continue looking for other invariants, edge cases, and properties not enumerated here.

## Schema
```jsonc
{
  "description": "<one-line property>",
  "function": "<function name or 'Contract-wide'>",
  "condition": "<precise property, citing enforcement sites by symbol name>",
  "path": "<contract path>"
}
```

# What this file is

- A list of properties the curators consider load-bearing for the protocol's correctness.
- Some entries also document intended behaviors that look like vulnerabilities on first read but are by design (e.g. externalShares writer enumeration, share-conservative transfer rounding). These are included as orientation, not as findings to flag.

# What this file is NOT

- Not exhaustive. The agent should derive additional invariants from the source where it sees properties that aren't documented here.
- Not a scope boundary. Findings outside the categories of these invariants are still in scope.


# How to use it

Read these first to anchor on properties that are known and intentional. Then look at source for anything the file doesn't cover. `condition` text cites enforcement sites by symbol name; resolve against live source.

---

# Sections

1. Core staking and tokens — Lido, StETH, StakeLimitUtils, NodeOperatorsRegistry.
2. Accounting, Burn and report execution — Accounting, Burner, OracleReportSanityChecker.
3. Withdrawals — WQ stack, WithdrawalVault, TriggerableWithdrawalsGateway, ExitLimitUtils.
4. Staking Router and Allocations — StakingRouter, MinFirstAllocationStrategy.


## 1. Core staking and tokens
```json 
[
  {
    "description": "clValidators ≤ depositedValidators (transient-ether term is non-negative).",
    "function": "processClStateUpdate",
    "condition": "After every processClStateUpdate and unsafeChangeDepositedValidators, the condition clValidators ≤ depositedValidators should hold. The assert(depositedValidators ≥ clValidators) inside _getInternalEther will brick all share-rate reads if violated.",
    "path": "contracts/0.4.24/Lido.sol"
  },
  {
    "description": "Buffered ether is always ≤ the contract's ETH balance (modulo uncounted direct transfers)",
    "function": "Contract-wide",
    "condition": "_getBufferedEther() ≤ address(this).balance at all times. Every credit to bufferedEther is gated by an actual ETH inflow (_submit requires msg.value != 0; receiveELRewards; receiveWithdrawals; rebalanceExternalEtherToInternal). Every debit matches an outflow (deposit sends exactly depositsCount·32 ETH to StakingRouter; collectRewardsAndProcessWithdrawals sends _etherToLockOnWithdrawalQueue). Overstating will result in deposit/finalize transfers reverting. While understating results in ether silently orphaned and excluded from the share rate.",
    "path": "contracts/0.4.24/Lido.sol"
  },
  {
    "description": "Each ETH inflow path is bijectively paired with a specific accounting update.",
    "function": "Contract-wide",
    "condition": "Three pairings: (1) default payable / submit: mint shares to msg.sender AND increment bufferedEther by msg.value. (2) receiveELRewards (_auth(_elRewardsVault)) : increment TOTAL_EL_REWARDS_COLLECTED by msg.value (buffer credited later, inside collectRewardsAndProcessWithdrawals during the report). (3) receiveWithdrawals (_auth(_withdrawalVault)) : event-only at inflow (buffer credited inside the report). Bijection must hold both ways: every accounting-update path requires a matching msg.value of the same magnitude, and every ETH-receiving path through Lido's own code must trigger its paired update.",
    "path": "contracts/0.4.24/Lido.sol"
  },
  {
    "description": "internalizeExternalBadDebt is the only path that lowers the share rate without removing shares, excluding fee-mint path.",
    "function": "Contract-wide",
    "condition": "internalizeExternalBadDebt(n) decreases externalShares by n, totalShares unchanged, internalEther unchanged. externalEther falls (the n shares stop being externally backed and the internal-share denominator rises by n, so the drop is strictly larger than getPooledEthByShares(n), approaching it only for n much smaller than internalShares), so totalPooledEther falls and the share rate drops for ALL holders. Auth-gated to _accounting(), it requires externalShares at least n. This is the protocol's mechanism for socializing vault-side losses, any non-Accounting caller path is a direct dilution attack on every stETH holder.",
    "path": "contracts/0.4.24/Lido.sol"
  },
  {
    "description": "The share-rate denominator (internalShares = totalShares − externalShares) is strictly positive and bounded by totalShares at all times.",
    "function": "Contract-wide",
    "condition": "internalShares = totalShares − externalShares is at least initialBootstrap, which is greater than zero. Maintained by construction: bootstrap mints to INITIAL_TOKEN_HOLDER (0xdead) at _bootstrapInitialHolder create internal shares only, and no code path transfers or burns them. The inline comment 'never 0 because of the stone in the elevator' inside _getShareRateDenominator is the protocol-wide assumption. Violation means division by zero in every share/ether conversion, so every transfer, submit, withdrawal, and rebase reverts.",
    "path": "contracts/0.4.24/Lido.sol"
  },
  {
    "description": "Sum of per-account shares equals totalShares after every state-changing operation.",
    "function": "Contract-wide",
    "condition": "The sum of shares[a] across every account a equals _getTotalShares() at all times. This is the core accounting identity the share rate prices against. If it breaks, every balanceOf, every withdrawal claim, and every reward computation reads a stale denominator.",
    "path": "contracts/0.4.24/StETH.sol"
  },
  {
    "description": "stETH balance is share pro-rata of internal ether",
    "function": "balanceOf",
    "condition": "balanceOf(a) == getPooledEthByShares(sharesOf(a)) == sharesOf(a) * _getShareRateNumerator() / _getShareRateDenominator() rounded down. In Lido the numerator is _getInternalEther() and the denominator is internalShares = totalShares - externalShares. Any divergence from this identity, is a violation.",
    "path": "contracts/0.4.24/StETH.sol"
  },
  {
    "description": "Submit mints shares at the pre-buffer-increase rate",
    "function": "_submit",
    "condition": "In _submit: sharesAmount = getSharesByPooledEth(msg.value) is computed BEFORE _setBufferedEther(_getBufferedEther() + msg.value); the buffer then increases by exactly msg.value and totalShares increases by exactly sharesAmount. Computing the share rate after the buffer grows (or minting a different amount) would let a submitter dilute existing holders or be diluted.",
    "path": "contracts/0.4.24/Lido.sol"
  },
  {
    "description": "Deposit moves ether from buffer to validators 1:1",
    "function": "deposit",
    "condition": "On a successful deposit() call with depositsCount > 0: bufferedEther_after == bufferedEther_before - depositsCount * DEPOSIT_SIZE, depositedValidators_after == depositedValidators_before + depositsCount, and depositsCount * DEPOSIT_SIZE <= getDepositableEther(). totalShares and totalPooledEther are unchanged by deposit because transientEther replaces consumed buffer.",
    "path": "contracts/0.4.24/Lido.sol"
  },
  {
    "description": "Total shares only changes via mint/burn paths",
    "function": "Contract-wide",
    "condition": "_getTotalShares() (TOTAL_AND_EXTERNAL_SHARES_POSITION low128) changes only inside _mintShares (submit, mintShares, mintExternalShares, fee minting during rebase, _mintInitialShares) and _burnShares (burnShares, burnExternalShares). Plain transfers, oracle reports, or any other flow, MUST NOT change totalShares. A rebase that changes user share balances or totalShares (other than the fee shares minted by Accounting) is a violation.",
    "path": "contracts/0.4.24/StETH.sol"
  }
]
```

## 2. Accounting, Burn and report execution
```json
[
  {
    "description": "stETH transfer is share-conservative, not ether-conservative",
    "function": "Contract-wide",
    "condition": "On StETH.transfer(to, amount): movedShares = getSharesByPooledEth(amount) (SafeMath.mul/div, rounds toward zero). The recipient's balanceOf increases by getPooledEthByShares(movedShares) ≤ amount; the sender's balance decreases by the same ≤ amount. The integer-division dust stays in the sender's share balance. Any code path that uses getPooledEthBySharesRoundUp to compute either side of a transfer, or that emits a Transfer(amount) event for a transfer that moved a non-matching share quantity, is a violation.",
    "path": "contracts/0.4.24/StETH.sol"
  },
  {
    "description": "The protocol-fee mintShares call is the last share-rate-changing operation in a report.",
    "function": "_applyOracleReportContext",
    "condition": "Within _applyOracleReportContext, Lido.mintShares for sharesToMintAsFees follows Burner.commitSharesToBurn and Lido.internalizeExternalBadDebt; no further mint, burn, or external-share write occurs before Lido.emitTokenRebase. The fee-distribution transferShares calls move shares without changing the share rate.",
    "path": "contracts/0.8.9/Accounting.sol"
  },
  {
    "description": "Bad debt internalized in a report does not exceed pre-report external shares.",
    "function": "_snapshotPreReportState / Lido.internalizeExternalBadDebt",
    "condition": "pre.badDebtToInternalize ≤ pre.externalShares; otherwise postExternalShares = pre.externalShares − pre.badDebtToInternalize underflows. Sourced from VaultHub without verification by Accounting.",
    "path": "contracts/0.8.9/Accounting.sol"
  },
  {
    "description": "Bad-debt internalization decrements VaultHub's counter and Lido's externalShares by the same amount within the same transaction.",
    "function": "_applyOracleReportContext",
    "condition": "When preBadDebtToInternalize > 0: VaultHub.decreaseInternalizedBadDebt(badDebt) inside _applyOracleReportContext is IMMEDIATELY followed by Lido.internalizeExternalBadDebt(badDebt). Both receive the same amount and both must succeed in the same tx (revert on either rolls back the other). If a vulnerability lets the two calls be issued separately, VaultHub's counter would advance while Lido's externalShares stayed inflated — the same bad debt becomes re-internalizable on the next report, double-applying dilution to every stETH holder. Any new code path that calls one without the other is a critical finding.",
    "path": "contracts/0.8.9/Accounting.sol"
  },
  {
    "description": "Cover shares are drained before non-cover in commitSharesToBurn",
    "function": "commitSharesToBurn",
    "condition": "sharesToBurnNowForCover = min(_sharesToBurn, coverSharesBurnRequested) is applied before any non-cover work. non-cover is only touched with the leftover _sharesToBurn - sharesToBurnNowForCover. If both buckets have pending shares, the non-cover counters change only after cover is exhausted. Draining non-cover first delays loss absorption and is a violation.",
    "path": "contracts/0.8.9/Burner.sol"
  },
  {
    "description": "Per-bucket burn accounting is conserved, no cross-bucket reclassification",
    "function": "_requestBurn / commitSharesToBurn",
    "condition": "For each bucket b in {cover, nonCover}: (lifetime Σ of _sharesAmount to _requestBurn with that _isCover) == totalBurnt_b + pendingRequested_b, where the counters are totalCoverSharesBurnt|totalNonCoverSharesBurnt and coverSharesBurnRequested|nonCoverSharesBurnRequested. _requestBurn writes only its own bucket; every commitSharesToBurn updates totalBurnt_b and pendingRequested_b for each bucket independently and by equal amounts (so a cover share is never charged to non-cover counters or vice versa). Any path that increments one bucket's totalBurnt without the matching decrement to that same bucket's pendingRequested is a violation.",
    "path": "contracts/0.8.9/Burner.sol"
  },
  {
    "description": "Negative CL rebase is bounded by the rolling slashing+penalty allowance",
    "function": "_checkCLBalanceDecrease / _askSecondOpinion",
    "condition": "When _preCLBalance > _postCLBalance + _withdrawalVaultBalance the diff = _preCLBalance - (_postCLBalance + _withdrawalVaultBalance) is appended to reportData via _addReportData(reportTimestamp, stakingRouterExitedValidators, diff). The rolling 18-day sum negativeCLRebaseSum = _sumNegativeRebasesNotOlderThan(reportTimestamp - 18 days) MUST satisfy negativeCLRebaseSum <= initialSlashingAmountPWei * ONE_PWEI * (postCLValidators - exitedValidatorsAt(t-18d)) + inactivityPenaltiesAmountPWei * ONE_PWEI * (postCLValidators - exitedValidatorsAt(t-54d)). If exceeded and secondOpinionOracle == address(0), revert IncorrectCLBalanceDecrease. If a second opinion is set, _askSecondOpinion enforces four guards in order: (1) success == true (else revert NegativeRebaseFailedSecondOpinionReportIsNotReady); (2) clBalanceWei >= _postCLBalance (else revert NegativeRebaseFailedCLBalanceMismatch — this guard also protects the subtraction in the next check from underflow); (3) MAX_BASIS_POINTS * (clBalanceWei - _postCLBalance) <= clBalanceOraclesErrorUpperBPLimit * clBalanceWei (else revert NegativeRebaseFailedCLBalanceMismatch); (4) oracleWithdrawalVaultBalanceWei == _withdrawalVaultBalance (else revert NegativeRebaseFailedWithdrawalVaultBalanceMismatch). Concrete falsifiers: widening the 18d/54d windows, replacing the per-validator scaling factors (postCLValidators - exitedAt(t-Xd)) with looser ones, dropping any of the four second-opinion guards, or removing the budget-exceeded revert when no second opinion is configured — each lets an oracle compromise inflate the protocol's accepted negative-rebase magnitude without on-chain confirmation.",
    "path": "contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol"
  },
  {
    "description": "Simulated share rate must stay within deviationBPLimit of the actual one",
    "function": "checkSimulatedShareRate / _checkSimulatedShareRate",
    "condition": "When report.withdrawalFinalizationBatches.length > 0: actualShareRate = (postInternalEther + etherToFinalizeWQ) * SHARE_RATE_PRECISION_E27 / (postInternalShares + sharesToBurnForWithdrawals); MAX_BASIS_POINTS * |actualShareRate - simulatedShareRate| / actualShareRate <= simulatedShareRateDeviationBPLimit; else revert IncorrectSimulatedShareRate. _noWithdrawalsPostInternalEther must be non-zero (asserted). A report whose simulatedShareRate diverges by more than the configured limit must be rejected — otherwise withdrawal request finalization is priced off the actual share rate.",
    "path": "contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol"
  },
  {
    "description": "prefinalize is view and its returned ETH/shares match what _finalize would lock",
    "function": "prefinalize / _finalize",
    "condition": "prefinalize(batches, simulatedShareRate) is declared `view` in WithdrawalQueueBase and reads only queue counters and the input batches. Its returned (ethToLock, sharesToBurn) must equal what _finalize would lock when later called with (_lastRequestIdToBeFinalized = batches[last], _amountOfETH = ethToLock, _maxShareRate = simulatedShareRate): sharesToBurn equals queue[last].cumulativeShares − queue[lastFinalizedRequestId].cumulativeShares; ethToLock = Σ over batches of either (cumulativeStETH delta) for nominal batches or (sharesDelta × simulatedShareRate / 1e27) for discounted batches. A code path that mutates queue state during prefinalize, or causes _finalize to lock/burn different totals than prefinalize predicted, would let Accounting's post-state math diverge from realized chain state.",
    "path": "contracts/0.8.9/WithdrawalQueueBase.sol"
  },
  {
    "description": "Protocol fees are minted only on a positive CL rewards delta",
    "function": "_calculateTotalProtocolFeeShares",
    "condition": "sharesToMintAsFees > 0 only if (report.clBalance + update.withdrawalsVaultTransfer) > update.principalClBalance (CL-only gate; elRewardsVaultTransfer does NOT enter it). The converse is not exact — with the gate open, a sub-precisionPoints totalRewards can still floor feeEther and sharesToMintAsFees to 0. Per LIP-12, a flat or negative CL delta MUST produce sharesToMintAsFees == 0. Any path that mints fee shares on a non-positive CL delta is a violation.",
    "path": "contracts/0.8.9/Accounting.sol"
  },
  {
    "description": "Fee distribution conserves shares: module fee shares plus treasury fee shares equal the total fee shares minted.",
    "function": "_calculateFeeDistribution",
    "condition": "moduleSharesToMint[i] = totalSharesAsFees × stakingModuleFees[i] / _totalFee (integer division, rounds down) and treasurySharesToMint = totalSharesAsFees − sum(moduleSharesToMint), so sum(moduleSharesToMint) + treasurySharesToMint = totalSharesAsFees. Treasury absorbs the per-module rounding dust.",
    "path": "contracts/0.8.9/Accounting.sol"
  },
  {
    "description": "Positive rebase per report is capped by maxPositiveTokenRebase",
    "function": "smoothenTokenRebase",
    "condition": "Let R = maxPositiveTokenRebase / 1e9. After every handleOracleReport: postInternalEther/postInternalShares ≤ (preInternalEther/preInternalShares) × (1 + R). Enforced by PositiveTokenRebaseLimiter inside smoothenTokenRebase: the limiter caps the running internal TPE at preInternalEther × (1 + R) and derives the share-burn budget from the same ceiling. The returned (withdrawals, elRewards, totalSharesToBurn) are the only inflow/burn amounts Accounting may apply. Any path that grows internal TPE or shrinks internal shares bypassing this gate is a violation.",
    "path": "contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol"
  },
  {
    "description": "Reported vault balances and burn-queue cannot exceed on-chain state",
    "function": "checkAccountingOracleReport",
    "condition": "Inside checkAccountingOracleReport: report.withdrawalVaultBalance ≤ address(withdrawalVault).balance (_checkWithdrawalVaultBalance); report.elRewardsVaultBalance ≤ address(elRewardsVault).balance (_checkELRewardsVaultBalance); report.sharesRequestedToBurn ≤ Burner.coverSharesBurnRequested + nonCoverSharesBurnRequested (_checkSharesRequestedToBurn). Each check reverts on violation. Any path that lets the report enter Accounting's apply phase with values larger than the corresponding on-chain reading is a violation — Accounting would pull more ETH than the vaults hold or commit more shares than queued.",
    "path": "contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol"
  },
  {
    "description": "Fee shares minted equal the self-consistent share-rate compensation amount",
    "function": "_calculateTotalProtocolFeeShares",
    "condition": "The LIP-12 profitability gate is CL-only: fee shares are minted only when (report.clBalance + withdrawalsVaultTransfer) > principalClBalance (a positive CL delta; elRewardsVaultTransfer does NOT enter the gate; even with the gate open, a sub-precisionPoints totalRewards can floor feeEther — and thus sharesToMintAsFees — to 0). When the gate passes: totalRewards = (report.clBalance + withdrawalsVaultTransfer) − principalClBalance + elRewardsVaultTransfer; feeEther = (totalRewards × totalFee) / precisionPoints; sharesToMintAsFees = (feeEther × internalSharesBeforeFees) / (postInternalEther − feeEther), where internalSharesBeforeFees already nets out the shares burned this report (= preInternalShares − totalSharesToBurn). The formula makes the post-mint internal share rate equal to (postInternalEther − feeEther) / internalSharesBeforeFees — i.e. recipients receive shares worth exactly feeEther at the new rate. A different formula either over- or under-pays fee recipients relative to the rewards they're entitled to.",
    "path": "contracts/0.8.9/Accounting.sol"
  },
  {
    "description": "_applyOracleReportContext executes its sub-steps in a fixed order with no interleaving",
    "function": "_applyOracleReportContext",
    "condition": "Within _applyOracleReportContext the calls execute in this exact order: (1) _sanityChecks; (2) burner.requestBurnShares for WQ-shares (if any); (3) LIDO.processClStateUpdate; (4) VaultHub.decreaseInternalizedBadDebt then LIDO.internalizeExternalBadDebt — both, in that order, if pre.badDebtToInternalize > 0; (5) burner.commitSharesToBurn; (6) LIDO.collectRewardsAndProcessWithdrawals; (7) LIDO.mintShares for sharesToMintAsFees; (8) _distributeFee (transferShares to recipients + treasury); (9) stakingRouter.reportRewardsMinted — steps (7)–(9) execute only if sharesToMintAsFees > 0; (10) _notifyRebaseObserver → postTokenRebaseReceiver.handlePostTokenRebase (only if registered); (11) LIDO.emitTokenRebase. No external call may be inserted between (1) and (10); no step may be reordered. Reordering changes the post-state math (e.g., fees computed against burned-but-not-finalized shares, or finalize priced against a post-mint rate), with no on-chain revert to signal the inconsistency.",
    "path": "contracts/0.8.9/Accounting.sol"
  }
]
```

## 3. Withdrawals
```json
[
  {
    "description": "TriggerableWithdrawalsGateway never retains user ETH: msg.value is fully consumed as totalFee plus refund",
    "function": "triggerFullWithdrawals / _checkFee / _refundFee",
    "condition": "msg.value >= requestsCount * getWithdrawalRequestFee() (else revert InsufficientFee); totalFee = requestsCount * fee is forwarded to WithdrawalVault.addWithdrawalRequests; refund = msg.value - totalFee is transferred to refundRecipient (or msg.sender if recipient == 0), reverting FeeRefundFailed if the transfer fails. Post-call address(this).balance equals its pre-call value (the preservesEthBalance assert holds). Any residual ETH left on TriggerableWithdrawalsGateway after the call is a violation.",
    "path": "contracts/0.8.9/TriggerableWithdrawalsGateway.sol"
  },
  {
    "description": "Exit rate-limit is consumed by exactly the batch size BEFORE any external call",
    "function": "triggerFullWithdrawals / _consumeExitRequestLimit",
    "condition": "When isExitLimitSet(): calculateCurrentExitLimit(now) >= validatorsData.length must hold. The consumption (prevExitRequestsLimit rewritten to currentLimit - validatorsData.length) runs BEFORE WithdrawalVault.addWithdrawalRequests and StakingRouter.onValidatorExitTriggered. Forwarding requests to the vault without first consuming the limit, or decrementing by an amount different from validatorsData.length, is a violation.",
    "path": "contracts/0.8.9/TriggerableWithdrawalsGateway.sol"
  },
  {
    "description": "Claim payout cannot exceed the request's nominal stETH amount",
    "function": "_claim / _calculateClaimableEther",
    "condition": "Let batchShareRate = (cumulativeStETH[id] - cumulativeStETH[id-1]) * E27_PRECISION_BASE / (cumulativeShares[id] - cumulativeShares[id-1]). If batchShareRate > checkpoint.maxShareRate: payout = shares * checkpoint.maxShareRate / E27_PRECISION_BASE (discounted); else payout = cumulativeStETH[id] - cumulativeStETH[id-1] (nominal). In both branches payout <= request.stETH at request time — a positive rebase between request and finalization never flows to the claimant. A claim returning more than the locked-at-request value breaks solvency.",
    "path": "contracts/0.8.9/WithdrawalQueueBase.sol"
  },
  {
    "description": "Locked ether is always covered by the contract balance and tracks finalized-unclaimed value",
    "function": "Contract-wide",
    "condition": "address(this).balance >= getLockedEtherAmount() at all times. _finalize increases lockedEther by exactly _amountOfETH (the msg.value of the finalize call). _claim decreases it by exactly the per-request payout. ETH paid to claimants must originate from balances credited by _finalize. Locked exceeding balance means a finalized claimant cannot be paid.",
    "path": "contracts/0.8.9/WithdrawalQueueBase.sol"
  },
  {
    "description": "A request can be claimed exactly once and only by its current owner",
    "function": "_claim",
    "condition": "Preconditions enforced: _requestId != 0 AND _requestId <= lastFinalizedRequestId AND !request.claimed AND request.owner == msg.sender. On success request.claimed becomes true and the id is removed from _getRequestsByOwner()[owner]. A second successful claim, or a claim by any address other than request.owner, is a violation (approvals grant transfer rights only, never claim rights).",
    "path": "contracts/0.8.9/WithdrawalQueueBase.sol"
  },
  {
    "description": "stETH custody is held by the WithdrawalQueue between request creation and finalization",
    "function": "Contract-wide",
    "condition": "Between _enqueue and _finalize, the queue contract holds the stETH backing each unfinalized request. Finalization is paired with a Burner share-burn so the locked stETH leaves the queue's balance only via burn or claim accounting, never to an arbitrary external recipient. STETH.balanceOf(address(this)) >= sum of (cumulativeStETH[id] - cumulativeStETH[id-1]) over id in (lastFinalizedRequestId, lastRequestId] at all times.",
    "path": "contracts/0.8.9/WithdrawalQueue.sol"
  },
  {
    "description": "Queue request IDs and cumulative counters are strictly monotone",
    "function": "_enqueue",
    "condition": "For every requestId r > 0 stored in WithdrawalQueueBase._getQueue()[r] (WithdrawalRequest struct defined in WithdrawalQueueBase): r is assigned r = getLastRequestId() + 1 inside _enqueue; after enqueue, queue[r].cumulativeStETH = queue[r-1].cumulativeStETH + amountOfStETH (strict increase since amountOfStETH ≥ MIN_STETH_WITHDRAWAL_AMOUNT = 100 wei); queue[r].cumulativeShares = queue[r-1].cumulativeShares + amountOfShares (strict increase). The last request id (read via getLastRequestId()) is monotonically non-decreasing and only mutated in _enqueue (via _setLastRequestId). The queue's sentinel at index 0 holds cumulativeStETH = 0, cumulativeShares = 0, owner = address(0), claimed = true, reportTimestamp = 0. Any code path that writes to an existing queue[r] — other than the claimed flag (set in _claim) or the owner field (reassigned by WithdrawalQueueERC721._transfer); neither touches the cumulative counters — or that assigns a non-sequential request id, breaks every cumulative-subtraction in prefinalize/_calcBatch/_calculateClaimableEther.",
    "path": "contracts/0.8.9/WithdrawalQueueBase.sol"
  }
]
```

## 4. Staking Router and Allocations
```json
[
  {
    "description": "Per-module deposit capacity respects stakeShareLimit and the module's available keys",
    "function": "_getDepositsAllocation",
    "condition": "Let T = Σ_i cache[i].activeValidatorsCount + _depositsToAllocate. For each i: capacities[i] == min( (cache[i].stakeShareLimit * T) / TOTAL_BASIS_POINTS, cache[i].activeValidatorsCount + cache[i].availableValidatorsCount ). allocate only ADDS to allocations[i] (seeded at cache[i].activeValidatorsCount) and skips any bucket with allocations[i] >= capacities[i], never adding beyond a capacity. So allocations[i] <= capacities[i] for every i EXCEPT a module already over its share target on entry (capacities[i] = targetValidators < activeValidatorsCount), which keeps its over-target count with 0 new deposits — not a violation. Real invariant: cache[i].activeValidatorsCount <= allocations[i] <= max(capacities[i], cache[i].activeValidatorsCount); only ADDING deposits beyond capacities[i] is a violation.",
    "path": "contracts/0.8.9/StakingRouter.sol"
  },
  {
    "description": "Distributed fee shares are conserved and zero-active-validator modules are omitted",
    "function": "getStakingRewardsDistribution",
    "condition": "Σ_j stakingModuleFees[j] (post-shrink) + treasuryFee == totalFee, where treasuryFee derives from totalFee - Σ stakingModuleFees (see getStakingFeeAggregateDistribution). Stopped modules contribute to totalFee via the treasuryFee summand but their stakingModuleFees[j] entry is 0 (forfeited to treasury). Modules with cache[i].activeValidatorsCount == 0 are skipped entirely and arrays are shrunk to rewardedStakingModulesCount; a consumer iterating beyond that length would misroute fees.",
    "path": "contracts/0.8.9/StakingRouter.sol"
  }
]
```