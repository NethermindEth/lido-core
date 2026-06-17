---
doc: "vault-flows"
title: Critical Flows — Supplementary Context (Vaults)
contracts: []
contracts: []
prereqs: []
see_also: ["00", "07"]
ssot_for: []
---
# Critical Flows — Supplementary Context (V3 stVaults)

> Accounting flows in the V3 Vaults system worth orienting around early. This file is **supplementary context, not a scope boundary** — it highlights important paths, but the agent should also examine flows, code paths, and edge cases not covered here. Use this as one lens among many.

## What's in this file
- Flow A — Mint / burn / health / force-rebalance (the solvency core).
- Flow B — Bad-debt socialization & internalization (Accounting seam).

Each entry has: *Why it matters* and *What it does* (numbered steps).

## What this file is NOT
- Not the canonical mechanics for any flow. Per-step mechanics live in the contract source.
- Not exhaustive. Other flows (vault lifecycle/connect, NAV report/quarantine, PDG deposit security, validator exits, governance confirm) are out of this file's frame and still in scope.
- Not an ordering signal. The "Flow A/B" naming is for reference inside this file only; it does not imply other paths deserve less scrutiny.

## How to use it
Read this first to orient on these accounting flows. Then go to the contract source for mechanics and everything else.

## Flow A: Mint, burn, health & force-rebalance (the solvency core)

### Why it matters
This is where external stETH is minted against vault collateral and where overcollateralization is enforced. Minting must stay within the tier reserve ratio; below the forced-rebalance threshold the vault is force-rebalanced. The `maxLiabilityShares` ratchet is the single guard against the *add ETH → mint → burn → apply stale report → withdraw freed collateral* cycle. A mis-priced `locked`/`withdrawable` lets an owner extract reserve-protected collateral.

### What it does
1. `fund` increments `inOutDelta` by the funded value [whenResumed, owner].
2. `mintShares` [owner (= Dashboard holding `MINT_ROLE`), whenResumed, fresh report]: gated at the VaultHub level by `whenResumed` + owner (`_checkConnectionAndOwner`) + fresh report — `MINT_ROLE` itself is enforced one layer up, in the Dashboard/`Permissions` contract that is the vault owner, not in VaultHub. `_increaseLiability` requires `sharesAfterMint ≤ connection.shareLimit` and `_locked(sharesAfter, minimalReserve, reserveRatioBP) ≤ _maxLockableValue`; ratchets `maxLiabilityShares`; enforces tier/group share caps via `OperatorGrid.onMintedShares` (`TierLimitExceeded`/`GroupLimitExceeded`), which also gates on the separate `VaultInJail` flag; mints external stETH.
3. `burnShares` [owner, whenResumed]: assumes the shares are already held by VaultHub (designed for smart-contract owners), then `_decreaseLiability` and burns external stETH. For EOA owners, `transferAndBurnShares` first pulls the shares in via `transferSharesFrom`, then calls `burnShares`.
4. Withdraw cap = `_withdrawableValue` = `min(availableBalance, totalValue) − redemptionValue`, capped at `unlocked = totalValue − locked`, minus `unsettledLidoFees`; `locked` uses `maxLiabilityShares` (the ratchet), not current `liabilityShares`.
5. `forceRebalance(vault)` [no role/owner gate, fresh report]: requires both `available = min(availableBalance, totalValue) > 0` (else `NoFundsForForceRebalance`) and outstanding obligations `_obligationsShares > 0` (else `NoReasonForForceRebalance`) — note this is *not* the `_obligationsShortfallValue` (= `obligationsAmount − balance`) that gates `forceValidatorExit`. Rebalances `min(_obligationsShares, sharesByPooledEth(available))` via `_decreaseLiability` + rebalance-external-to-internal. `_obligationsShares = max(healthShortfallShares, redemptionShares)`; `healthShortfallShares == type(uint256).max` when `liability > totalValue` (bad debt is unfixable by rebalance — `forceRebalance` still consumes the vault's available funds, but the residual shortfall beyond them must be cleared via socialize/internalize).

## Flow B: Bad-debt socialization & internalization (Accounting seam)

### Why it matters
When there is a drops in vault's `totalValue` below its minted liability, the shortfall is **bad debt**. The resolution paths are DAO-role-gated: socialize the liability to a same-operator vault, or internalize it to the protocol. Internalization dilutes **every** stETH holder, so the bad-debt counter and the external-shares decrement must stay paired or the books desync (the same loss could be applied twice).

### What it does
1. `_badDebtShares = liabilityShares − sharesByPooledEth(totalValue)` when positive.
2. `socializeBadDebt(badDebtVault, acceptor, maxShares)` [`BAD_DEBT_MASTER_ROLE`, fresh reports]: requires `nodeOperator(acceptor) == nodeOperator(badDebtVault)`; moves `min(badDebt, maxShares, acceptorCapacity)` liability from donor to acceptor (`acceptorCapacity = acceptorTotalValueShares − acceptorLiabilityShares`).
3. `internalizeBadDebt(badDebtVault, maxShares)` [`BAD_DEBT_MASTER_ROLE`]: `_decreaseLiability`; `badDebtToInternalize += n` (cached at the current refSlot).
4. During the next oracle report, the accounting flow reads `badDebtToInternalizeForLastRefSlot()`, then (same tx) calls `decreaseInternalizedBadDebt(d)` and the core `internalizeExternalBadDebt(d)` — folding `d` external shares into internal, dropping the internal share rate.
