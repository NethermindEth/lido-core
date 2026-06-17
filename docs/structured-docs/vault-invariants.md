---
doc: "vault-invariants"
title: Invariants — V3 stVaults
contracts: []
prereqs: ["AGENTS-2"]
see_also: ["00", "07"]
ssot_for: []
---
# Invariants — Supplementary Properties (V3 stVaults)

> A curated set of **objective, accounting-related** properties worth verifying in the V3 stVaults system, expressed as machine-readable invariants. This file is **supplementary context, not the full audit-target set** — treat these as known properties to keep in mind, and continue looking for other invariants, edge cases, and properties not enumerated here.

## Schema
```jsonc
{
  "description": "<one-line property>",
  "function": "<function name or 'Contract-wide'>",
  "condition": "<precise property, with symbol refs to enforcement sites>",
  "path": "<contract path>"
}
```

# What this file is

- A list of accounting properties the curators consider load-bearing for the vaults' correctness, each verifiable against the source.

# What this file is NOT

- Not exhaustive. Derive additional invariants from source where you see properties not documented here.
- Not a scope boundary. Findings outside the categories below are still in scope.

# How to use it

Read these to anchor on known objective properties, then read source for anything not covered. `condition` text uses symbol names; resolve against live source.

---

# Sections

1. Liability, value & withdraw cap — `VaultHub`, `StakingVault`.
2. Bad debt & Accounting seam — `VaultHub` ↔ `Accounting`/`Lido`.
3. System-wide solvency conservation — `VaultHub` ↔ `Lido`/`OperatorGrid`.

## 1. Liability, value & withdraw cap
```json
[
  {
    "description": "locked is computed from maxLiabilityShares, not current liabilityShares.",
    "function": "_locked",
    "condition": "locked = getPooledEthBySharesRoundUp(maxLiabilityShares) + max(reserve, minimalReserve); reserve = ceilDiv(liability * reserveRatioBP, TOTAL_BASIS_POINTS - reserveRatioBP); minimalReserve = max(CONNECT_DEPOSIT, slashingReserve). Computing locked from current liabilityShares instead of maxLiabilityShares re-opens the add-ETH -> mint -> burn -> apply-stale-report -> withdraw collateral-unlock cycle.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  },
  {
    "description": "maxLiabilityShares ratchets up on mint and only drops on a report with no post-refSlot minting.",
    "function": "mintShares / applyVaultReport",
    "condition": "On mint, maxLiabilityShares = max(maxLiabilityShares, sharesAfterMint). In applyVaultReport it is lowered to max(liabilityShares, reportLiabilityShares) only when record.maxLiabilityShares == reportMaxLiabilityShares (nothing minted on funds added after the refSlot). An unconditional lowering of maxLiabilityShares breaks the withdraw-cap ratchet.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  },
  {
    "description": "Mint is bounded by both the share limit and the lockable-value (reserve) limit.",
    "function": "_increaseLiability",
    "condition": "sharesAfterMint = liabilityShares + amount must satisfy sharesAfterMint <= shareLimit AND _locked(sharesAfterMint, minimalReserve, reserveRatioBP) <= _maxLockableValue(record, 0), where _maxLockableValue = totalValue - unsettledLidoFees. Either check missing allows under-collateralized minting.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  },
  {
    "description": "Withdrawable value never exceeds available balance, unlocked value, or net-of-fees value.",
    "function": "_withdrawableValue",
    "condition": "withdrawable = min(availableBalance, totalValue); minus redemptionValue (0 if redemptionValue > that); then min with unlocked = totalValue - locked; then minus unsettledLidoFees (floored at 0). withdraw(vault, recipient, ether) requires ether <= withdrawable and ether <= StakingVault.availableBalance(). A withdraw exceeding this lets reserve-protected or redemption-earmarked collateral leave.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  },
  {
    "description": "availableBalance excludes staged ETH; staged ETH is not withdrawable.",
    "function": "availableBalance",
    "condition": "StakingVault.availableBalance() == address(this).balance - stagedBalance. Staged ETH (reserved for in-flight PDG activations) is excluded from withdraw and from the withdraw caps. Counting staged ETH as available lets in-flight activation funds be withdrawn.",
    "path": "contracts/0.8.25/vaults/StakingVault.sol"
  },
  {
    "description": "Margin-changing operations require a fresh report.",
    "function": "_isReportFresh",
    "condition": "_isReportFresh = uint48(latestReportTimestamp) <= record.report.timestamp AND block.timestamp - latestReportTimestamp < REPORT_FRESHNESS_DELTA (2 days). Required by mintShares/withdraw/rebalance/forceRebalance/forceValidatorExit/socialize/internalize (via _requireFreshReport). A stale report cannot be minted/withdrawn against.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  },
  {
    "description": "Margin-reducing operations require a fresh report; margin-improving operations deliberately do not.",
    "function": "mintShares / withdraw / rebalance vs burnShares / fund",
    "condition": "mintShares, withdraw, rebalance, forceRebalance, forceValidatorExit and socialize/internalize call _requireFreshReport. burnShares and fund (debt-down / collateral-in, i.e. margin-improving) intentionally do NOT. No margin-reducing path (debt up or collateral out) may execute against a stale value; both adding freshness to burn/fund and removing it from mint/withdraw are regressions.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  },
  {
    "description": "Share->ETH conversions in collateral checks round up, so rounding can only over-collateralize.",
    "function": "_locked / _isThresholdBreached / _getPooledEthBySharesRoundUp",
    "condition": "locked and the health/threshold comparison convert liabilityShares to ETH via getPooledEthBySharesRoundUp (round UP), and reserve uses Math256.ceilDiv. Rounding error can therefore only increase locked / tighten the threshold, never release collateral. Switching any of these to round-down under-collateralizes the vault.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  },
  {
    "description": "Vault health is defined on CURRENT liabilityShares against the forced-rebalance threshold (distinct from locked, which uses the maxLiabilityShares ratchet).",
    "function": "_isThresholdBreached / _isVaultHealthy / _healthShortfallShares",
    "condition": "healthy iff getPooledEthBySharesRoundUp(liabilityShares) <= totalValue * (TOTAL_BASIS_POINTS - forcedRebalanceThresholdBP) / TOTAL_BASIS_POINTS. This is the force-rebalance trigger and the basis of healthShortfallShares. NB: health uses current liabilityShares whereas _locked uses maxLiabilityShares; do not conflate the two.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  }
]
```

## 2. Bad debt & Accounting seam
```json
[
  {
    "description": "Bad debt is the uncollateralized share shortfall.",
    "function": "_badDebtShares",
    "condition": "_badDebtShares = liabilityShares - getSharesByPooledEth(totalValue) when liabilityShares > totalValueShares, else 0. This is the figure socialize/internalize operate on; it is derived from oracle totalValue.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  },
  {
    "description": "socializeBadDebt moves at most the acceptor's capacity and requires a same node-operator acceptor.",
    "function": "socializeBadDebt",
    "condition": "socializeBadDebt [BAD_DEBT_MASTER_ROLE, fresh reports] requires nodeOperator(acceptor) == nodeOperator(badDebtVault); it moves min(badDebt, maxShares, acceptorCapacity) liability from donor to acceptor, where acceptorCapacity = acceptorTotalValueShares - acceptorLiabilityShares (0 if negative). Moving more than acceptorCapacity, or to a different-operator acceptor, is a violation.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  },
  {
    "description": "Internalized bad debt accrues to a refSlot-cached counter, decremented only by Accounting.",
    "function": "internalizeBadDebt / decreaseInternalizedBadDebt",
    "condition": "internalizeBadDebt [BAD_DEBT_MASTER_ROLE] does _decreaseLiability then badDebtToInternalize += n (RefSlotCache.withValueIncrease). decreaseInternalizedBadDebt is gated to LIDO_LOCATOR.accounting(). badDebtToInternalize() returns the live value; badDebtToInternalizeForLastRefSlot() returns the value cached at the last refSlot.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  },
  {
    "description": "Bad-debt resolution relocates debt; it never creates or destroys net debt.",
    "function": "socializeBadDebt / internalizeBadDebt",
    "condition": "liabilityShares decreased on the donor vault == liabilityShares increased on the acceptor (socialize) OR == the increment to badDebtToInternalize (internalize). SUM(liabilityShares) + badDebtToInternalize is conserved across either transition, and neither path mints or burns core external shares (LIDO supply unchanged).",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  }
]
```

## 3. System-wide solvency conservation
```json
[
  {
    "description": "Global solvency identity: every external stETH share is backed by a vault liability or the internalized bad-debt counter.",
    "function": "Contract-wide (mintShares / burnShares / _rebalance / socializeBadDebt / internalizeBadDebt / decreaseInternalizedBadDebt)",
    "condition": "total core external stETH shares minted via vaults == SUM over connected vaults of record.liabilityShares + badDebtToInternalize.value. mint/burn move core supply (LIDO.mintExternalShares/burnExternalShares) and a vault's liabilityShares together; rebalance reduces both; socialize relocates liability between vaults (supply unchanged); internalize moves liability into badDebtToInternalize (supply unchanged) until Accounting burns it and calls decreaseInternalizedBadDebt. No reachable state leaves stETH unbacked AND unaccounted.",
    "path": "contracts/0.8.25/vaults/VaultHub.sol"
  }
]
```
