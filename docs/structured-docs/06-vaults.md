---
doc: "06"
title: Vaults & PDG
contracts: [StakingVault, VaultHub, VaultFactory, PinnedBeaconProxy, PinnedBeaconUtils, RefSlotCache, TriggerableWithdrawals, OperatorGrid, LazyOracle, Confirmable2Addresses, Dashboard, Permissions, NodeOperatorFee, AccessControlConfirmable, Confirmations, PredepositGuarantee, CLProofVerifier, MeIfNobodyElse]
prereqs: []
see_also: ["00","03","07","R"]
ssot_for: [vault-health, force-rebalance, withdraw-cap, pdg-state-machine, vault-roles, consolidation, bad-debt-internalize-seam]
---
# 06 — Vaults & PDG

> Lido V3 stVaults: isolated, non-custodial staking positions that mint stETH as **external shares** against their own ETH backing. Core Pool seam: `Lido.mintExternalShares`/`burnExternalShares`/`rebalanceExternalEtherToInternal` ([01](./01-core-staking.md#core-flows)); bad-debt write-off lands in Accounting ([03](./03-oracle-accounting.md#core-flows)); CL proof + consolidation model in [R](./R-consensus-proof-reference.md#distilled-external-specs).

## Sub-index

(roles per contract in the Contracts table; flows below.)
- **vaults-core** — `StakingVault`, `VaultHub`, `VaultFactory`, `PinnedBeaconProxy`/`PinnedBeaconUtils`, `RefSlotCache`, `TriggerableWithdrawals`.
- **economics** — `OperatorGrid` (tiers/groups), `LazyOracle` (NAV report + quarantine), `Confirmable2Addresses` (owner+operator dual-confirm).
- **governance** — `Dashboard`, `Permissions`/`AccessControlConfirmable`/`Confirmations` (role + multi-confirm spine), `NodeOperatorFee`.
- **PDG** — `PredepositGuarantee` (1-ETH bond + proof), `CLProofVerifier` (SSZ/EIP-4788 WC proof), `MeIfNobodyElse` (self-guarantor sentinel).

## Contracts

| Contract | File | Role |
|---|---|---|
| `StakingVault` | `0.8.25/vaults/StakingVault.sol` | Per-vault ETH custody; `0x02` WC = `bytes32(WC_0X02_PREFIX \| uint160(address(this)))`. Behind `PinnedBeaconProxy`. |
| `VaultHub` | `0.8.25/vaults/VaultHub.sol` | Registry + accounting hub: `VaultConnection`/`VaultRecord`, liability shares, health, withdraw cap, force-rebalance, bad debt; mints/burns external stETH. |
| `VaultFactory` | `0.8.25/vaults/VaultFactory.sol` | Deploys `StakingVault + Dashboard` pairs; sole connect allowlist (`deployedVaults`). |
| `PinnedBeaconProxy` | `0.8.25/vaults/PinnedBeaconProxy.sol` | `BeaconProxy` that can pin (ossify) the current impl, opting out of beacon upgrades. |
| `PinnedBeaconUtils` | `0.8.25/vaults/lib/PinnedBeaconUtils.sol` | `ossify()` writes current impl into `PINNED_BEACON_STORAGE_SLOT`; `AlreadyOssified` if repeated. |
| `RefSlotCache` | `0.8.25/vaults/lib/RefSlotCache.sol` | `Uint104WithCache` + `DoubleRefSlotCache.Int104WithCache` (signed); caches value + value-at-refSlot keyed by HashConsensus refSlot. |
| `TriggerableWithdrawals` | `common/lib/TriggerableWithdrawals.sol` | EIP-7002 exit/withdrawal encoder; sole importer is `StakingVault`. |
| `OperatorGrid` | `0.8.25/vaults/OperatorGrid.sol` | Tier/group params (share limit, reserve ratio, fees); per-operator usage; jail; dual-confirm tier moves. |
| `LazyOracle` | `0.8.25/vaults/LazyOracle.sol` | V3 NAV oracle: Merkle root + per-vault proof; quarantines sudden positive value jumps. |
| `Confirmable2Addresses` | `0.8.25/utils/Confirmable2Addresses.sol` | Two-address mutual-confirm base over `Confirmations` (owner+operator as confirmers); `OperatorGrid` mixes it in. |
| `Dashboard` | `0.8.25/vaults/dashboard/Dashboard.sol` | Staker-facing object that owns the `StakingVault` and proxies every hub-mediated call. |
| `Permissions` | `0.8.25/vaults/dashboard/Permissions.sol` | Role primitive for `Dashboard`; `renounceRole` disabled; mass grant/revoke with per-role admin. |
| `NodeOperatorFee` | `0.8.25/vaults/dashboard/NodeOperatorFee.sol` | Operator-fee accrual on vault growth + claim/correction accounting. |
| `AccessControlConfirmable` | `0.8.25/utils/AccessControlConfirmable.sol` | `AccessControlEnumerable` + confirm; resolves confirmers by `hasRole`; base of `Permissions`. |
| `Confirmations` | `0.8.25/utils/Confirmations.sol` | Multi-party confirm keyed by exact calldata, with expiry; base of the above. |
| `PredepositGuarantee` (PDG) | `0.8.25/vaults/predeposit_guarantee/PredepositGuarantee.sol` | 1-ETH-bonded predeposit + CL proof before the 31-ETH activation. |
| `CLProofVerifier` | `0.8.25/vaults/predeposit_guarantee/CLProofVerifier.sol` | SSZ Merkle proof of validator (pubkey, WC) against an EIP-4788 beacon root. |
| `MeIfNobodyElse` | `0.8.25/vaults/predeposit_guarantee/MeIfNobodyElse.sol` | Self-pointer sentinel: unset → return the key itself (operator is own guarantor/depositor). |
| `Accounting` (boundary, [03](./03-oracle-accounting.md)) | `0.8.9/Accounting.sol` | Calls `VaultHub.decreaseInternalizedBadDebt` as treasury makes bad debt whole. |
| `Lido` (boundary, [01](./01-core-staking.md)) | `0.4.24/Lido.sol` | `mintExternalShares`/`burnExternalShares`/`rebalanceExternalEtherToInternal` Core-Pool seam. |

Two distinct "owners": (a) the `StakingVault` `Ownable2Step` owner — `Dashboard` at deploy, then **VaultHub** after `connectVault`, so every `onlyOwner` vault fn routes through VaultHub once connected; (b) `VaultConnection.owner` in VaultHub's registry — the **Dashboard** (enforced by `_checkConnectionAndOwner`). The staker holds `DEFAULT_ADMIN_ROLE` on the Dashboard.

**Deployment state (mainnet).** The V3 singletons (`VaultHub`, `LazyOracle`, `OperatorGrid`, `PredepositGuarantee`) are **already deployed and initialized** on mainnet ([`deployed-mainnet.json`](../../deployed-mainnet.json)); `VaultFactory`, the beacon, and the shared `StakingVault`/`Dashboard` implementations are deployed alongside them (the impls are logic templates — per-vault instances are initialized on demand, flow 1). The singletons sit behind `OssifiableProxy`, whose constructor delegatecalls the encoded `initialize(...)` **inside the deploy transaction** (no uninitialized window; the `initializer` guard is spent at construction), and each singleton's implementation constructor calls `_disableInitializers()` so the logic contract can never be initialized directly. `VaultFactory` is non-upgradeable; singleton-impl upgrades are a separate Agent action via the proxy admin's `proxy__upgradeTo*` ([`07`](./07-governance-permissions.md#core-flows)). **`VaultHub` and `PredepositGuarantee` are constructed paused** (`_pauseUntil(PAUSE_INFINITELY)`), so their `whenResumed`-gated flows below — VaultHub's fund/mint/burn/withdraw, connect, and report application, plus PDG's predeposit/prove/activate — need a `resume()` by a `RESUME_ROLE` holder before they operate; `LazyOracle` and `OperatorGrid` are not pausable this way.

## Core flows

### 1. Creation and connect

```text
staker → VaultFactory.createVaultWithDashboard{value ≥ VaultHub.CONNECT_DEPOSIT}(admin, nodeOperator, noManager, feeBP, confirmExpiry, roles)
  → new PinnedBeaconProxy(BEACON) ⇒ StakingVault         // deployedByThisFactory[vault]=true
  → Clones.cloneWithImmutableArgs(DASHBOARD_IMPL, vault) ⇒ Dashboard (ERC-1167)
  → StakingVault.initialize(owner=Dashboard, nodeOperator, depositor=locator.predepositGuarantee())
  → Dashboard.initialize(...) → Dashboard.connectToVaultHub{value}() → VaultHub.connectVault(vault)
      require deployedVaults(vault) ∧ depositor==PDG ∧ availableBalance ≥ CONNECT_DEPOSIT
      require stagedBalance == PDG.pendingActivations(vault) * ACTIVATION_DEPOSIT_AMOUNT (31 ETH)
      → OperatorGrid.vaultTierInfo → connection params; disconnectInitiatedTs = type(uint48).max
      → StakingVault.acceptOwnership()                   // EXT: VaultHub becomes Ownable2Step owner
  → Dashboard grants DEFAULT_ADMIN_ROLE to admin, revokes its own
External: VaultFactory is the sole allowlist (`PREVIOUS_FACTORY` chains older factories).
```

Alt `createVaultWithDashboardWithoutConnectingToVaultHub(...)` deploys/inits but skips `connectToVaultHub` (no `CONNECT_DEPOSIT`); Dashboard stays vault owner, only operator-side roles auto-granted; connect later. The `connectVault` guards are load-bearing: a vault with tampered pre-connect storage cannot connect (`deployedByThisFactory` is private, no admin override), and the `depositor==PDG` + `stagedBalance==pendingActivations*31 ETH` checks ensure PDG mediates all deposits and no activation ETH is unaccounted.

### 2. Fund, withdraw, mint, burn

```text
Dashboard.fund() [FUND_ROLE] → VaultHub.fund(vault) [whenResumed, owner] → StakingVault.fund() [onlyOwner=VaultHub]   // inOutDelta += value
Dashboard.mint{Shares,StETH,WstETH} [MINT_ROLE] → VaultHub.mintShares(vault, recipient, shares) [whenResumed, owner, fresh report]
  → _increaseLiability: sharesAfterMint ≤ connection.shareLimit; _locked(sharesAfter, minimalReserve, reserveRatioBP) ≤ _maxLockableValue
  → maxLiabilityShares = max(maxLiabilityShares, sharesAfterMint)        // ratchet (see mechanics)
  → OperatorGrid.onMintedShares (tier/group caps; VaultInJail reverts)
  → Lido.mintExternalShares(recipient, shares)                          // EXT: Core Pool totalShares++
Dashboard.burn{Shares,StETH,WstETH} [BURN_ROLE] → transferSharesFrom(sender→VaultHub) → VaultHub.burnShares(vault, shares) [whenResumed, owner]
  → _decreaseLiability → Lido.burnExternalShares(shares)                // EXT
```

`burnShares` burns shares already held by VaultHub (for contract owners); EOA owners use `transferAndBurnShares` (does `transferSharesFrom` then `burnShares`, so the owner check fires inside `burnShares`). Vault liability is denominated in **shares**, so when stETH rebases up the ETH liability grows too, eroding health — intentional; the reserve ratio absorbs it.

### 3. Health, withdraw cap, force-rebalance (load-bearing)

The solvency core. `liability(shares) = getPooledEthBySharesRoundUp(shares)`. A vault is **unhealthy** when `_isThresholdBreached(totalValue, liabilityShares, forcedRebalanceThresholdBP)`, i.e. `liability(liabilityShares) > totalValue * (10000 − forcedRebalanceThresholdBP) / 10000`. `totalValue = report.totalValue + inOutDelta.currentValue() − report.inOutDelta` (live between reports).

```text
withdraw cap (NOT totalValue − locked − obligationsShortfall):
  available  = min(availableBalance(vault), totalValue)                 // availableBalance = balance − stagedBalance
  step1      = available − redemptionValue        ; if redemptionValue > available ⇒ 0   (redemptionValue = pooledEthRoundUp(redemptionShares))
  feesIncl   = min(step1, unlocked)               ; unlocked = totalValue − locked (floored at 0)
  withdrawable = feesIncl − unsettledLidoFees     (floored at 0)        // = _withdrawableValue
Dashboard.withdraw [WITHDRAW_ROLE] → VaultHub.withdraw(vault, recipient, ether) [whenResumed, owner, fresh report]
  require ether ≤ withdrawable → StakingVault.withdraw(recipient, ether) [onlyOwner=VaultHub; also ether ≤ availableBalance()]
```

`locked = liability(maxLiabilityShares) + max(reserve, minimalReserve)`, `reserve = ceilDiv(liability * reserveRatioBP, 10000 − reserveRatioBP)`, `minimalReserve = max(CONNECT_DEPOSIT, slashingReserve)`. **The ratchet:** `locked` uses `maxLiabilityShares`, not current `liabilityShares`, blocking the one-tx exploit *add ETH → mint → burn → apply old report → withdraw freed collateral*: `maxLiabilityShares` drops only on a report, and only when `record.maxLiabilityShares == report.maxLiabilityShares` (never if anything was minted on funds added after the refSlot — see mechanics). `isVaultHealthy()` uses *current* `liabilityShares` + `forcedRebalanceThresholdBP`; `locked()` uses `maxLiabilityShares` + `reserveRatioBP` + `minimalReserve` — two predicates.

```text
forceRebalance(vault)  — PERMISSIONLESS (no role, no owner check); fresh report
  available = min(availableBalance, totalValue) ; revert NoFundsForForceRebalance if 0
  shares = min(_obligationsShares, sharesByPooledEth(available)) ; revert NoReasonForForceRebalance if 0
  → _rebalance: _decreaseLiability(shares); _withdraw(VaultHub, pooledEthRoundUp(shares)); Lido.rebalanceExternalEtherToInternal{value}(shares)  // EXT
```

The unhealthy state is the authorization — no role gate. `_obligationsShares = max(healthShortfallShares, redemptionShares)`; `healthShortfallShares` solves `(L−X)/(TV−X) = (10000−reserveRatioBP)/10000` for the ETH `X` to repay, `+100` shares rounding safety, capped at `liabilityShares` (returns `type(uint256).max` when `liability > totalValue`, i.e. bad debt). In that state `forceRebalance` does **not** no-op: with available ETH on hand it executes and drains it (down to sub-share dust), debiting `liability` and `totalValue` by the same value — so it cannot restore health and does **not reduce** the shortfall (`_badDebtShares` can only grow by ≤1 share of rounding, never shrink); the residual must be cleared via socialize/internalize (flow 9). (`available == 0` ⇒ `NoFundsForForceRebalance`; `available` worth < 1 share ⇒ `NoReasonForForceRebalance`.) Voluntary deleveraging is `rebalance(vault, shares)` [owner, fresh report]; force-rebalance does **not** settle Lido fees. `settleLidoFees(vault)` [permissionless, fresh report] sends `min(withdrawableFeesIncluded, unsettledFees)` to treasury.

### 4. Validator exits and force-exit (load-bearing)

```text
VaultHub.requestValidatorExit(vault, pubkeys) [owner] → StakingVault.requestValidatorExit → emits events only (off-chain operators honor)
VaultHub.triggerValidatorWithdrawals(vault, pubkeys, amountsInGwei, refundRecipient) [payable, owner]
  → if any partial (amount>0): if isVaultInJail(vault) ⇒ PartialValidatorWithdrawalNotAllowed; fresh report; if obligationsShortfall>0 ⇒ PartialValidatorWithdrawalNotAllowed
  → StakingVault.triggerValidatorWithdrawals [onlyOwner=VaultHub, payable]
      → TriggerableWithdrawals.add{Full,}WithdrawalRequests → WITHDRAWAL_REQUEST predeploy   // EXT: EIP-7002
VaultHub.forceValidatorExit(vault, pubkeys, refundRecipient) [payable, VALIDATOR_EXIT_ROLE, fresh report]
  → require obligationsShortfallValue > 0 (ForcedValidatorExitNotAllowed); amountsInGwei = [] (full exit)
  → StakingVault.triggerValidatorWithdrawals → TriggerableWithdrawals → WITHDRAWAL_REQUEST predeploy   // EXT: EIP-7002, NOT TriggerableWithdrawalsGateway
StakingVault.ejectValidators(pubkeys, refundRecipient) [payable, msg.sender==nodeOperator] → TriggerableWithdrawals (full) — operator-side force-exit
```

Force-exit routes **directly** through `StakingVault → TriggerableWithdrawals → WITHDRAWAL_REQUEST` predeploy — **not** through `TriggerableWithdrawalsGateway` (the Core-Pool path in [04](./04-withdrawals.md#core-flows)). Partial withdrawals are blocked while any obligations shortfall is uncovered — and entirely for jailed vaults — so an owner cannot clog the CL queue by front-running the forced full exits needed to rebalance. CL processing of the emitted EIP-7002 full/partial requests — the skip/cap/queue ladder `process_withdrawal_request` applies — is in [`R`](./R-consensus-proof-reference.md#consensus-layer-request-processing-the-seam).

### 5. PDG state machine (load-bearing)

Threat: a malicious operator deposits a vault's 32 ETH to a validator carrying *someone else's* WC, stealing it. PDG risks only `PREDEPOSIT_AMOUNT = 1 ETH` per validator until a CL proof confirms the validator's WC equals the vault's `0x02` WC.

```text
1. predeposit(deposits[]) [msg.sender == _depositorOf(nodeOperator)]   // 1 ETH/validator
   → balance.locked += 1 ETH * n (consumes pre-funded bond: requires unlocked = total − locked ≥ n ETH) ; pendingActivations[vault] += n
   → StakingVault.depositToBeaconChain (vault WC)   // EXT: deposit contract ; status PREDEPOSITED
   → StakingVault.stage(31 ETH * n)                 // sets aside, not withdrawable
2. validator becomes pending on CL
3a. proveWCAndActivate(witness) — PERMISSIONLESS; CLProofVerifier proves (pubkey, WC) at a beacon root
    → WC matches ⇒ unlock 1 ETH ; StakingVault.depositFromStaged(31 ETH) ⇒ status ACTIVATED  // EXT: deposit contract
3b. proveInvalidValidatorWC(...) — PERMISSIONLESS; proof shows WC mismatch
    → locked 1 ETH forfeit, sent to the vault (PREDEPOSITED ⇒ COMPENSATED); pendingActivations −−
3c. proveUnknownValidator(witness, vault) [msg.sender == vault.owner()] — register a side-deposited validator (NONE ⇒ ACTIVATED)
```

`predeposit` guards: `EmptyDeposits`, depositor gate, and per deposit `PredepositAmountInvalid` (must equal `PREDEPOSIT_AMOUNT` = 1 ETH), `ValidatorNotNew`, and BLS `verifyDepositMessage` — the on-chain proof the predeposited validator is genuine (not someone else's key).

Bond plumbing (all `PredepositGuarantee`): `topUpNodeOperatorBalance(nodeOperator)` [payable, guarantor]; **`withdrawNodeOperatorBalance(nodeOperator, amount, recipient)`** [`onlyGuarantorOf`]; `amount` must be a multiple of `PREDEPOSIT_AMOUNT` and only *unlocked* bond; `setNodeOperatorGuarantor`/`setNodeOperatorDepositor` (who funds / who deposits); `claimGuarantorRefund(recipient)`. Reentrancy: `pendingActivations` is decremented **before** the external call in the activate/top-up path.

`MeIfNobodyElse` is the self-guarantor sentinel: `getValueOrKey(map, key)` returns `key` when `map[key] == address(0)`, and `setOrReset(map, key, value)` stores `address(0)` when `value == key`. So an operator with no explicit guarantor/depositor is, by default, its own guarantor and depositor (typical for sole-operator vaults) without a storage write.

### 6. CL proof verification (CLProofVerifier, load-bearing)

`CLProofVerifier._validatePubKeyWCProof(witness, withdrawalCredentials)` walks an SSZ Merkle branch from the validator record up to the beacon state root (`GI_STATE_ROOT`; validator GIndex by fork-pivot `GI_FIRST_VALIDATOR_PREV`/`_CURR`), then checks it against the EIP-4788 `BEACON_ROOTS` predeploy via `staticcall(abi.encode(childBlockTimestamp))` in `_getParentBlockRoot`. Same primitive as `ValidatorExitDelayVerifier` ([08](./08-exits.md#core-flows)), but PDG verifies *WC at deposit* while the exit verifier verifies *exit status*. Proof model + GIndex/SSZ math in [R](./R-consensus-proof-reference.md#core-flows).

### 7. NAV report and quarantine (LazyOracle)

```text
AccountingOracle → LazyOracle.updateReportData(timestamp, refSlot, treeRoot, cid)   [msg.sender==locator.accountingOracle(), NO role]
anyone → LazyOracle.updateVaultData(vault, totalValue, cumulativeLidoFees, liabilityShares, maxLiabilityShares, slashingReserve, proof)
  → MerkleProof.verify(proof, treeRoot, leaf) else InvalidProof    // authenticity is the proof, not the caller
  → sanity: timestamp strictly increases; maxLiabilityShares ≥ liabilityShares else InvalidMaxLiabilityShares (no upper bound — may exceed record.maxLiabilityShares);
            cumulativeLidoFees non-decreasing, Δ ≤ maxLidoFeeRatePerSecond*elapsed; totalValue ≤ uint96 max; no inOutDelta underflow
  → if reportedTotalValue > onchainRefSlotValue * (10000 + maxRewardRatioBP)/10000 ⇒ QUARANTINE (positive jumps only; flat %, NOT time-scaled)
  → else fold in immediately (negative jumps always apply at once — slashing is real)
  → VaultHub.applyVaultReport(vault, ...)   // updates report, cumulativeLidoFees, minimalReserve, maxLiabilityShares (guarded)
VaultHub → LazyOracle.removeVaultQuarantine(vault)   [msg.sender==locator.vaultHub()]   // cleared on disconnect finalization (inside VaultHub.applyVaultReport)
UPDATE_SANITY_PARAMS_ROLE → updateSanityParams(quarantinePeriod ≤ MAX_QUARANTINE_PERIOD=30d, maxRewardRatioBP ≤ MAX_REWARD_RATIO=type(uint16).max, maxLidoFeeRatePerSecond ≤ MAX_LIDO_FEE_RATE_PER_SECOND=10 ETH/s)
```

Quarantine bounds a malicious/buggy reporter to "any positive jump, but wait the `quarantinePeriod` delay before it counts." `fund()` increases are tracked via `inOutDelta`, not the report, so they bypass quarantine. The threshold is relative to vault value, so the same absolute off-book gain may be normal for a large vault but quarantined for a small one.

### 8. Tiers (OperatorGrid) and dual-confirm

```text
REGISTRY_ROLE → registerGroup / updateGroupShareLimit / registerTiers / alterTiers / updateVaultFees(→VaultHub.updateConnection)
              / setConfirmExpiry / setVaultJailStatus
owner|operator → changeTier(vault, tierId, shareLimit) | syncTier(vault) | updateVaultShareLimit(vault, limit)
  → Confirmable2Addresses dual-confirm (owner + operator addresses); returns false until both confirm within confirmExpiry
  → rejects requestedShareLimit < liabilityShares; checks tier/group caps → VaultHub.updateConnection(vault, params…) [fresh report]
VaultHub → onMintedShares (VaultInJail / TierLimitExceeded / GroupLimitExceeded) | onBurnedShares | resetVaultTier (→ DEFAULT_TIER_ID=0)   [caller==vaultHub]
```

`effectiveShareLimit(vault) = min(connection shareLimit, tier remaining + liability, group remaining + liability)`. Jailed vaults cannot mint (burn/rebalance still work). `OperatorGrid._validateParams` (every tier — default at `initialize`, plus `registerTiers`/`alterTiers`) requires `reserveRatioBP ∈ [1, 9999]` (`MAX_RESERVE_RATIO_BP`, ≤99.99%), `forcedRebalanceThresholdBP` non-zero with `+10 bp < reserveRatioBP` (`ForcedRebalanceThresholdTooHigh`), and `infra/liquidity/reservationFeeBP ≤ MAX_FEE_BP = type(uint16).max` (≈655% — tier fees are **not** capped at 100%). Separately, `VaultHub._requireSaneShareLimit` (connect / `updateConnection`) caps `shareLimit ≤ getTotalShares() × MAX_RELATIVE_SHARE_LIMIT_BP / TOTAL_BASIS_POINTS` — a TVL-relative ceiling (immutable, non-zero, `≤ 10000`; `ShareLimitTooHigh`). Tier params bound each vault's overcollateralization; the owner+operator dual-confirm here is *addresses-as-confirmers*, distinct from the role-based `Permissions` confirm in flow 10.

### 9. Bad debt (load-bearing seam to Accounting)

`_badDebtShares = liabilityShares − sharesByPooledEth(totalValue)` when positive (liability exceeds what `totalValue` backs after a slash). Both resolutions are `BAD_DEBT_MASTER_ROLE` + fresh report:

```text
socializeBadDebt(badDebtVault, acceptor, maxShares)  — require nodeOperator(acceptor)==nodeOperator(badDebt)
  → move min(badDebt, maxShares, acceptorCapacity) liability from donor to acceptor (no burn, no negative rebase)
internalizeBadDebt(badDebtVault, maxShares)
  → _decreaseLiability(badDebt, n); badDebtToInternalize += n (RefSlotCache.withValueIncrease)   // protocol absorbs
decreaseInternalizedBadDebt(shares) [msg.sender==locator.accounting()]  → badDebtToInternalize.value -= shares
```

`socializeBadDebt` applies the moved liability to the acceptor with limits overridden (`_overrideOperatorLimits=true`, `shareLimit`/`lockableValueLimit=max` — reserve/tier/group/jail enforcement skipped), bounded by `acceptorCapacity = max(0, acceptorTotalValueShares − acceptorLiabilityShares)` (the acceptor's unencumbered backing — the only cap the overrides leave). The acceptor stays out of *formal* bad debt (`_badDebtShares`→0 at any accepted amount), but a large enough move pushes its liability past `forcedRebalanceThresholdBP` — **unhealthy** (at the extreme, an `acceptorCapacity`-bound move drives liability to its full backing) — making it a permissionless `forceRebalance` (+ `VALIDATOR_EXIT_ROLE`-gated `forceValidatorExit`) target (flows 3–4) whose *own* ETH repays the donor's loss. The **only** guard is `nodeOperator(acceptor)==nodeOperator(badDebt)` — **no acceptor-owner consent** — so `BAD_DEBT_MASTER_ROLE` (`VAULTS_ADAPTER`, [07](./07-governance-permissions.md#role-matrix-high-impact-roles-only)) can aim it at any healthy same-operator vault; and since it checks `_requireConnected` (not `_checkConnection`), socializing onto a vault mid voluntary-disconnect (at `liabilityShares==0`) re-raises liability and **aborts that disconnect** at the next report (flow 12). `internalizeBadDebt` writes the loss to `badDebtToInternalize`; `Accounting` consumes it during the report (treasury makes the Core Pool whole) and calls `decreaseInternalizedBadDebt` — the seam to [03](./03-oracle-accounting.md#core-flows). Between `internalizeBadDebt` and the next report, `externalShares` ≠ sum of vault liabilities; the bucket bridges the gap.

**Accounting-side settlement (seam — a vault audit needs only this, not the rest of [`03`](./03-oracle-accounting.md#core-flows)).** Driven by the oracle report: at snapshot `Accounting` reads `VaultHub.badDebtToInternalizeForLastRefSlot()` (the permissionless `simulateOracleReport` twin reads the live `badDebtToInternalize()` instead — the two can differ, so the daemon simulates against the current ref-slot view). Then, before the share burn/finalize, if `badDebt > 0`:
```text
Accounting._applyOracleReportContext (if badDebtToInternalize > 0):
  → VaultHub.decreaseInternalizedBadDebt(d)        // debits this bucket (flow 9)
  → Lido.internalizeExternalBadDebt(d)             // d shares external → internal; folds postExternalShares -= d, postInternalShares += d ⇒ internal rate drops
```
The two calls must stay **paired** or the books desync (the `decreaseInternalizedBadDebt` ↔ `internalizeExternalBadDebt` double-settle surface).

### 10. Per-vault governance (Dashboard) and confirm spine

`Dashboard` owns the vault while disconnected and proxies every hub call once connected. The split: **staker** (`DEFAULT_ADMIN_ROLE`, admins staker-side roles) controls capital — `FUND`/`WITHDRAW`/`MINT`/`BURN`/`REBALANCE`, `VAULT_CONFIGURATION`, beacon-deposit pause, exit/trigger, `VOLUNTARY_DISCONNECT`, `COLLECT_VAULT_ERC20`; **node operator** (`NODE_OPERATOR_MANAGER_ROLE`, own admin of the three operator sub-roles) controls fee + `NODE_OPERATOR_UNGUARANTEED_DEPOSIT`/`_PROVE_UNKNOWN_VALIDATOR`/`_FEE_EXEMPT`. Neither can grant itself the other's powers. `Permissions.renounceRole` disabled. Lifecycle: `connectToVaultHub` → `_transferOwnership(VAULT_HUB)`; `voluntaryDisconnect` collects operator fee into `feeLeftover`, stops accrual → `VaultHub.voluntaryDisconnect`; `abandonDashboard(newOwner)` (disconnected only); `reconnectToVaultHub` (needs `settledGrowth` correction if `feeRate > 0`); `transferVaultOwnership(newOwner)` is **dual-confirm** (`DEFAULT_ADMIN_ROLE` + `NODE_OPERATOR_MANAGER_ROLE`) → `VaultHub.transferVaultOwnership` (reassigns registry owner *without* disconnecting). `Dashboard.setPDGPolicy` sets `PDGPolicy` ∈ {`STRICT` (default), `ALLOW_PROVE`, `ALLOW_DEPOSIT_AND_PROVE`}; the two PDG-bypass entrypoints are `unguaranteedDepositToBeaconChain` (direct `depositContract.deposit`, skipping the 1-ETH predeposit+proof — `NODE_OPERATOR_UNGUARANTEED_DEPOSIT_ROLE`, requires `ALLOW_DEPOSIT_AND_PROVE`) and `proveUnknownValidatorsToPDG` (`NODE_OPERATOR_PROVE_UNKNOWN_VALIDATOR_ROLE`, forbidden under `STRICT`); default `STRICT` forbids both.

Confirm spine `Dashboard → NodeOperatorFee → Permissions → AccessControlConfirmable → Confirmations`: `Confirmations._collectAndCheckConfirmations(msg.data, confirmingRoles())` records one confirmation per role keyed by **exact calldata**, executing only once **all** roles confirm within `confirmExpiry` (default 1 day, bounded `MIN_CONFIRM_EXPIRY = 1 hours` .. `MAX_CONFIRM_EXPIRY = 30 days` — no `confirmExpiry` *constant*). A caller holding all roles passes in one tx. Used by `transferVaultOwnership`, `setFeeRate`, `correctSettledGrowth`, `setConfirmExpiry`, OperatorGrid tier/share-limit ops.

### 11. Operator fee (NodeOperatorFee)

`growth = (report.totalValue + quarantineValue) − report.inOutDelta`; `accruedFee = max(0, growth − settledGrowth) * feeRate / 10000`. `disburseFee()` is permissionless but reverts `AbnormallyHighFee` when fee > 1% of `(totalValue + quarantineValue)`; only `disburseAbnormallyHighFee` (`DEFAULT_ADMIN_ROLE`) bypasses. `setFeeRate` (dual-confirm, fresh report, disburses at old rate first), `correctSettledGrowth` (dual-confirm CAS), `addFeeExemption` (`NODE_OPERATOR_FEE_EXEMPT_ROLE`, raises `settledGrowth` so an amount — unguaranteed deposits, consolidated principal — is not fee-charged), `setFeeRecipient`. Quarantine value is in the fee base before VaultHub credits it; mis-sized exemptions shift value between owner and operator.

### 12. Disconnect and ossify

`voluntaryDisconnect(vault)` [owner, fresh report] sets `disconnectInitiatedTs = block.timestamp` (settles fees in full — reverts if balance can't cover). `disconnect(vault)` [`VAULT_MASTER_ROLE`] same path, best-effort settlement. Completion happens **later inside `applyVaultReport`** (LazyOracle-only) when `liabilityShares == 0 ∧ slashingReserve == 0`: VaultHub `transferOwnership` back to `connection.owner`, releases `CONNECT_DEPOSIT`, `_deleteVault`; else aborted/deferred. `StakingVault.ossify()` [owner] → `PinnedBeaconUtils.ossify()` pins the current impl permanently (`AlreadyOssified` if repeated); `connectVault` rejects an ossified vault. `renounceOwnership` reverts.

## Internal mechanics

- **VaultHub storage.** `VaultConnection{owner, uint96 shareLimit, uint96 vaultIndex (0=not connected), uint48 disconnectInitiatedTs (type(uint48).max=connected, 0=disconnected, else pending), uint16 reserveRatioBP, forcedRebalanceThresholdBP, infraFeeBP, liquidityFeeBP, reservationFeeBP, bool beaconChainDepositsPauseIntent}`. `VaultRecord{Report report{uint104 totalValue, int104 inOutDelta, uint48 timestamp}, uint96 maxLiabilityShares, uint96 liabilityShares, DoubleRefSlotCache.Int104WithCache[2] inOutDelta, uint128 minimalReserve, uint128 redemptionShares, uint128 cumulativeLidoFees, uint128 settledLidoFees}`.
- **maxLiabilityShares ratchet (CEI / accounting hazard).** On mint, `maxLiabilityShares = max(maxLiabilityShares, sharesAfterMint)`. In `applyVaultReport` it drops to `max(liabilityShares, reportLiabilityShares)` only **when `record.maxLiabilityShares == reportMaxLiabilityShares`** (nothing minted on funds added after the refSlot). Single guard against the mint→burn→stale-report→withdraw collateral-unlock cycle (flow 3).
- **Report freshness.** `_isReportFresh` = `latestReportTimestamp ≤ record.report.timestamp ∧ block.timestamp − latestReportTimestamp < REPORT_FRESHNESS_DELTA (2 days)`. Required by mint/withdraw/rebalance/force-rebalance/force-exit; stale reports cannot be minted against.
- **totalValue & inOutDelta.** `totalValue = report.totalValue + inOutDelta.currentValue() − report.inOutDelta`. `inOutDelta` is the signed sum of all `fund − withdraw`, in the signed double `RefSlotCache` so a report reads the value frozen at its refSlot, not the live one. `badDebtToInternalize` uses the unsigned single cache (`getValueForLastRefSlot`).
- **Obligations.** `_obligationsShares = max(healthShortfallShares, redemptionShares)`; `_obligationsAmount = pooledEthRoundUp(obligationsShares) + (unsettledFees if ≥ MIN_BEACON_DEPOSIT)`; `_obligationsShortfallValue = obligationsAmount − availableBalance` (floored). Beacon deposits stay paused while a shortfall is uncovered, even after the manual pause flag is cleared.
- **StakingVault custody.** `availableBalance = balance − stagedBalance`; staged ETH (reserved for in-flight activations) is excluded from `withdraw`. `stage`/`depositToBeaconChain` are `whenDepositsNotPaused`; `depositFromStaged(deposit, additionalAmount)` with `additionalAmount == 0` is **not** blocked by the deposit pause (by design — lets staged activations finish) but reverts if `additionalAmount > 0` while paused. `withdrawalCredentials()` is always `0x02 || 11×0 || address(this)`. `setDepositor` is `onlyOwner` (`NewDepositorSameAsPrevious`); `collectERC20` recovers stray ERC20 but reverts on the EIP-7528 ETH sentinel.
- **PinnedBeaconProxy.** `isOssified() = getPinnedImplementation() != address(0)`; `_implementation()` returns the pinned impl if set, else the beacon's. `PINNED_BEACON_STORAGE_SLOT = keccak256("stakingVault.proxy.pinnedBeacon") − 1`. All vaults share one `UpgradeableBeacon` (`stakingVaultBeacon`, owner = Agent), so a single `UpgradeableBeacon.upgradeTo` re-points **every non-pinned vault at once**; `StakingVault.ossify()` (flow 12) pins a vault's current impl and opts it out.
- **Cross-contract vault-keyed state on disconnect.** VaultHub `connections`/`records` cleared (`_deleteVault`); LazyOracle quarantine cleared (`removeVaultQuarantine`); OperatorGrid tier reset to `DEFAULT_TIER_ID` (liabilities must already be 0); **NodeOperatorFee** `feeRate` **preserved** but `settledGrowth` **force-set to `MAX_SANE_SETTLED_GROWTH`** in the same Dashboard proxy — so reconnect needs `correctSettledGrowth` when `feeRate>0`; **PDG** `pendingActivations`/status **not** force-cleared — reconnect re-checks `stagedBalance == pendingActivations * 31 ETH`.
- **Share-rate dependence.** `locked`, `isVaultHealthy`, `healthShortfallShares`, `badDebtShares` all use live `getPooledEthBySharesRoundUp`/`getSharesByPooledEth`, so a Core-Pool share-rate move changes a vault's ETH-side collateralization with no vault write.
- **Dashboard token handling.** wstETH moves via OZ `SafeERC20` (`safeTransfer`/`safeTransferFrom`, with `WSTETH.wrap`/`unwrap`); stETH is **never** SafeERC20-transferred — `Dashboard` wraps minted stETH immediately and otherwise uses `IStETH` share methods (`transferShares`/`transferSharesFrom`) and conversions (`getPooledEthBySharesRoundUp`), and never holds an stETH balance.

## External interactions

```text
VaultHub
  ← Dashboard (owner): fund, withdraw, mint/burn(+transferAndBurn)Shares, rebalance, requestValidatorExit,
      triggerValidatorWithdrawals, pause/resumeBeaconChainDeposits, voluntaryDisconnect, transferVaultOwnership,
      proveUnknownValidatorToPDG, collectERC20FromVault
  ← anyone: forceRebalance, settleLidoFees, connectVault
  ← VALIDATOR_EXIT_ROLE: forceValidatorExit ; BAD_DEBT_MASTER_ROLE: socialize/internalizeBadDebt
  ← REDEMPTION_MASTER_ROLE: setLiabilitySharesTarget ; VAULT_MASTER_ROLE: disconnect
  ← LazyOracle: applyVaultReport ; OperatorGrid: updateConnection ; Accounting: decreaseInternalizedBadDebt
  → StakingVault.fund/withdraw/requestValidatorExit/triggerValidatorWithdrawals/collectERC20/pause·resume/ownership (VaultHub is owner once connected)
  → EXT Lido.mintExternalShares / burnExternalShares / rebalanceExternalEtherToInternal   (Core Pool, see 01)
  → PredepositGuarantee.proveUnknownValidator ; LazyOracle.removeVaultQuarantine ; OperatorGrid.onMinted/onBurned/resetVaultTier
StakingVault
  ← owner (=VaultHub connected; Dashboard before): fund/withdraw/ownership/setDepositor/ossify
  ← depositor (PDG): stage/unstage/depositToBeaconChain/depositFromStaged ; ← nodeOperator: ejectValidators
  → EXT IDepositContract.deposit (via depositor) ; EXT EIP-7002 WITHDRAWAL_REQUEST predeploy via TriggerableWithdrawals (NOT the Gateway)
PredepositGuarantee
  ← nodeOperator/depositor (predeposit, topUpExistingValidators) ; ← guarantor (topUpNodeOperatorBalance, withdrawNodeOperatorBalance, claimGuarantorRefund)
  ← anyone (proveWCAndActivate, proveInvalidValidatorWC) ; ← vault owner / VaultHub (proveUnknownValidator)
  → CLProofVerifier → EXT BEACON_ROOTS precompile (EIP-4788) ; → StakingVault deposit/stage paths
LazyOracle ← AccountingOracle (updateReportData) / anyone+proof (updateVaultData) / VaultHub (removeVaultQuarantine) / UPDATE_SANITY_PARAMS_ROLE → VaultHub.applyVaultReport
OperatorGrid ← REGISTRY_ROLE / owner+operator (dual-confirm) / VaultHub → VaultHub.updateConnection
```

> **Invariants:** see [`vault-invariants.md`](./vault-invariants.md) — supplementary, not the full set; derive others from source.

## Key constants

| Constant | Value | Purpose |
|---|---|---|
| `CONNECT_DEPOSIT` | 1 ETH | Locked on connect, released on finalized disconnect; anti-spam / oracle-load floor; not mintable collateral. |
| `REPORT_FRESHNESS_DELTA` | 2 days | Mint/withdraw/rebalance/force require a report within this window. |
| `TOTAL_BASIS_POINTS` | 10000 | Reserve-ratio / threshold denominator. |
| `WC_0X02_PREFIX` | `0x02 << 248` | `0x02` withdrawal-credentials prefix (compounding type). |
| `PREDEPOSIT_AMOUNT` | 1 ETH | PDG bonded predeposit per validator. |
| `ACTIVATION_DEPOSIT_AMOUNT` | 31 ETH | Top-up to complete a 32-ETH activation after the 1-ETH predeposit. |
| `MAX_TOPUP_AMOUNT` | `2048 ETH − 31 − 1` | Per-validator top-up cap (EIP-7251 `0x02` compounding max effective balance = 2048 ETH). |
| `MAX_QUARANTINE_PERIOD` | 30 days | Upper bound on LazyOracle quarantine. |
| `MAX_REWARD_RATIO` / `MAX_LIDO_FEE_RATE_PER_SECOND` | `type(uint16).max` / 10 ETH/s | LazyOracle report-sanity caps (`updateSanityParams`, `UPDATE_SANITY_PARAMS_ROLE`). |
| `MAX_RELATIVE_SHARE_LIMIT_BP` | ≤ 10000 (deploy-set immutable) | TVL-relative ceiling on a connection `shareLimit` (`VaultHub._requireSaneShareLimit`; `ShareLimitTooHigh`). |
| `MIN_CONFIRM_EXPIRY` / `MAX_CONFIRM_EXPIRY` | 1 hour / 30 days | Confirm-window bounds (default 1 day); no `confirmExpiry` constant. |
| `DEFAULT_TIER_ID` | 0 | Tier a vault resets to on disconnect. |
| `WITHDRAWAL_REQUEST` | `0x…007002` | EIP-7002 vault exit/withdrawal predeploy (`TriggerableWithdrawals`). |
| `BEACON_ROOTS` | `0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02` | EIP-4788 beacon-root precompile (CLProofVerifier). |
| EIP-7251 consolidation predeploy | `0x0000BBdDc7CE488642fb579F8B00f3a590007251` | Consolidation request system contract (out-of-scope encoder `ValidatorConsolidationRequests`; see below). |

**EIP-7251 (consolidation) — distilled** (full facts in [R](./R-consensus-proof-reference.md#distilled-external-specs)). A vault consolidates source validators into a target via the predeploy `0x…007251` with 96-byte `source‖target` calldata; fee via `staticcall("")`. `source == target` with the validator's own `0x01` WC switches it to `0x02` (compounding), raising the effective-balance cap from 32 ETH (`MIN_ACTIVATION_BALANCE`) toward 2048 ETH (`MAX_EFFECTIVE_BALANCE_ELECTRA`) — the 2048 the vault's `MAX_TOPUP_AMOUNT` derives from. On-chain encoder is `ValidatorConsolidationRequests` (CLI-side, out-of-scope); it encodes inline and does **not** use `TriggerableWithdrawals` (EIP-7002-only). CL caveat: rewards above the source's effective balance sweep to the *source* WC, not the target — hence the paired `NodeOperatorFee.addFeeExemption`. CL processing — the switch-to-compounding vs. real-consolidation skip ladder `process_consolidation_request` applies — is distilled in [`R`](./R-consensus-proof-reference.md#consensus-layer-request-processing-the-seam).

## Source references

**Live source** (every symbol cited inline above resolves against these files):
- `0.8.25/vaults/`: `StakingVault.sol` (custody, exits, ossify), `VaultHub.sol` (`VaultConnection`/`VaultRecord`, connect/mint/burn, `_withdrawableValue`/`_locked`, force-rebalance, health, bad-debt, `applyVaultReport`), `VaultFactory.sol`, `PinnedBeaconProxy.sol`, `lib/PinnedBeaconUtils.sol`, `lib/RefSlotCache.sol`, `OperatorGrid.sol`, `LazyOracle.sol`.
- `0.8.25/vaults/dashboard/`: `Dashboard.sol`, `Permissions.sol`, `NodeOperatorFee.sol` (role split, lifecycle, fee accrual). `0.8.25/utils/`: `Confirmations.sol`, `AccessControlConfirmable.sol`, `Confirmable2Addresses.sol` (`MIN`/`MAX_CONFIRM_EXPIRY`, confirm spine).
- `0.8.25/vaults/predeposit_guarantee/`: `PredepositGuarantee.sol` (bond + state machine, `withdrawNodeOperatorBalance(nodeOperator, amount, recipient)`), `CLProofVerifier.sol` (`BEACON_ROOTS`, GIndex/SSZ proof), `MeIfNobodyElse.sol`. `common/lib/TriggerableWithdrawals.sol` (`WITHDRAWAL_REQUEST`).

**Official docs (`docs/`):** `docs/lido-v3-whitepaper.mdx` + `Lido_V3_Whitepaper.pdf` (overcollateralization, reserve ratio, health, tiers, PDG); `run-on-lido/stvaults/features-and-mechanics/parameters-and-metrics.md`, `roles-and-permissions.md`; `.../operational-and-management-guides/health-monitoring-guide.md`.