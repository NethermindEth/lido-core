---
doc: "07"
title: Governance & Permissions
contracts: [LidoLocator, OssifiableProxy, GateSeal]
prereqs: []
see_also: ["00"]
ssot_for: [role-matrix, ossification, gateseal, dual-governance]
---
# 07 — Governance and Permissions

> Cross-cutting authority layer over all 39 in-scope contracts (owns 0 of them). It answers "who can call this, and what is the blast radius if that holder is compromised." Per-flow roles are named at their flow step in the owning module (e.g. [`03`](./03-oracle-accounting.md), [`06`](./06-vaults.md)); this doc is the derived cross-contract index plus the proxy/ossification/locator authority seam.

## Contracts

| Contract | File | Role |
|---|---|---|
| `LidoLocator` | `0.8.9/LidoLocator.sol` | Protocol-wide service locator and **trust anchor**: 22 peer addresses baked into impl bytecode as `immutable`s (immutable-immutables). Behind `OssifiableProxy`. Every caller-identity auth resolves through it. |
| `OssifiableProxy` | `0.8.9/proxy/OssifiableProxy.sol` | ERC-1967 proxy with an admin slot that can be ossified (admin → `address(0)`, permanent). Backs the locator, oracles, router, Burner, Accounting, VaultHub, etc. |
| `GateSeal` | external (Vyper, separate repo) | One-shot, expiring, bounded emergency pause holder for `PausableUntil`-style targets. Holds `PAUSE_ROLE`; cannot resume. |
| Aragon `AGENT` | external (Aragon) | OZ-core admin + proxy admin for the V3 contracts; the on-chain root authority, itself governed by Dual Governance. |
| DG `Timelock`/`Executor`, `Escrow`, `ResealManager` | external (`lidofinance/dual-governance`) | Veto-timelock interposed before the Agent; escrow drives the state machine; ResealManager extends a GateSeal pause while DG is non-Normal. |

Three permissioning systems are layered: (1) **Aragon ACL** on legacy 0.4.24 (`Lido`, `StETH`, `NodeOperatorsRegistry`); (2) **OZ `AccessControl`/`AccessControlEnumerable`** on 0.8.x; (3) the **`owner`** single-address pattern (`DepositSecurityModule`, the `OssifiableProxy` admin slot). All three root in the Aragon `AGENT`, governed by DG.

## Role matrix (high-impact roles only)

> **Derived cross-contract index — not the source of truth.** A navigation aid only. The SSOT for any role is the inline mention at its flow step in the owning per-contract module. **On any conflict, the cited per-contract module wins.** Holders below are the post-V3 final ACL: the OZ-`AccessControl` rows on `Burner`, `VaultHub`, `OperatorGrid`, `LazyOracle`, `AccountingOracle`, `OracleReportSanityChecker`, `Accounting`, `PredepositGuarantee` and `StakingRouter` are the ones `V3Template._assertFinalACL` actually asserts; the remaining rows (Aragon-ACL `Lido`/`NodeOperatorsRegistry` roles, `WithdrawalQueue` operational roles, `DepositSecurityModule.owner`, etc.) are sourced from the owning per-contract module, not from the template assertion.

| Contract | Role | Holder (final ACL) | Notes |
|---|---|---|---|
| `Lido` (Aragon) | `PAUSE_ROLE` / `RESUME_ROLE` | Agent, governed by DG | Halt / resume everything |
| `Lido` | `STAKING_PAUSE_ROLE` / `STAKING_CONTROL_ROLE` | Agent, governed by DG | Stop submissions / set stake-rate limit |
| `Lido` | `UNSAFE_CHANGE_DEPOSITED_VALIDATORS_ROLE` | Agent, governed by DG | Dangerous manual counter override |
| `NodeOperatorsRegistry` | `MANAGE_NODE_OPERATOR_ROLE` | Agent / Easy Track | Add/deactivate operators |
| `NodeOperatorsRegistry` | `SET_NODE_OPERATOR_LIMIT_ROLE` | Agent → Easy Track factory | Vet keys (routine, optimistic) |
| `NodeOperatorsRegistry` | `STAKING_ROUTER_ROLE` | `StakingRouter` | SR drives the module |
| `StakingRouter` | `DEFAULT_ADMIN_ROLE` | Agent, governed by DG | Grants other roles |
| `StakingRouter` | `MANAGE_WITHDRAWAL_CREDENTIALS_ROLE` | Agent, governed by DG | Rotate protocol WC |
| `StakingRouter` | `STAKING_MODULE_MANAGE_ROLE` | Agent, governed by DG | Add/update modules |
| `StakingRouter` | `STAKING_MODULE_UNVETTING_ROLE` | `DepositSecurityModule` | Unvet via guardian quorum |
| `StakingRouter` | `REPORT_EXITED_VALIDATORS_ROLE` | `AccountingOracle` | Report exited counts |
| `StakingRouter` | `REPORT_REWARDS_MINTED_ROLE` | `Accounting` | Inform modules of fee mints |
| `StakingRouter` | `REPORT_VALIDATOR_EXIT_TRIGGERED_ROLE` | `TriggerableWithdrawalsGateway` | EIP-7002 hook |
| `HashConsensus` | `MANAGE_MEMBERS_AND_QUORUM_ROLE` / frame / processor roles | Agent, governed by DG | Committee + frame config |
| `AccountingOracle` / VEBO | `DEFAULT_ADMIN_ROLE`, `MANAGE_CONSENSUS_*` | Agent, governed by DG | Re-point consensus / bump version |
| `OracleReportSanityChecker` | `DEFAULT_ADMIN_ROLE` + all 12 limit-manager roles | Agent admin; **all limit roles unassigned** | V3 leaves limit-tuners empty (asserted zero holders) |
| `WithdrawalQueue` | `PAUSE_ROLE` / `RESUME_ROLE` | GateSeal + ResealManager / ResealManager | Emergency halt |
| `WithdrawalQueue` | `FINALIZE_ROLE` | `Lido` (via Accounting) | Finalize during report |
| `WithdrawalQueue` | `ORACLE_ROLE` | `AccountingOracle` | onOracleReport (bunker) |
| `TriggerableWithdrawalsGateway` | `ADD_FULL_WITHDRAWAL_REQUEST_ROLE` | `ValidatorsExitBusOracle` | Trigger exits |
| `Burner` | `REQUEST_BURN_SHARES_ROLE` | **`Accounting` + `CSM_ACCOUNTING` only** | Pre-approved share burns |
| `Burner` | `REQUEST_BURN_MY_STETH_ROLE` | Agent / Insurance fund | Voluntary burn |
| `Burner` | `DEFAULT_ADMIN_ROLE`, proxy admin | Agent, governed by DG | Recovery fns are permissionless (no role) |
| `VaultHub` | `VALIDATOR_EXIT_ROLE` / `BAD_DEBT_MASTER_ROLE` | `VAULTS_ADAPTER` | full vault set in [`06`](./06-vaults.md) |
| `VaultHub` | `REDEMPTION_MASTER_ROLE` / `VAULT_MASTER_ROLE` | unassigned (zero holders) | asserted empty in V3 |
| `VaultHub`/`PredepositGuarantee` | `PAUSE_ROLE` / `RESUME_ROLE` | GateSeal + ResealManager / ResealManager | Emergency halt |
| `OperatorGrid` | `REGISTRY_ROLE` | `EVM_SCRIPT_EXECUTOR` + `VAULTS_ADAPTER` | Easy Track + adapter |
| `DepositSecurityModule` | `owner` | Agent, governed by DG | full admin (single address) |

**`REQUEST_BURN_SHARES_ROLE` holders.** Exactly `Accounting` and `CSM_ACCOUNTING` — `V3Template._assertFinalACL` sets `holders[0]=ACCOUNTING; holders[1]=CSM_ACCOUNTING` and asserts via `_assertOZRoleHolders`. `Lido`/`WithdrawalQueue`/`VaultHub` do **not** hold it (on the *new* Burner it is granted to `ACCOUNTING` + `CSM_ACCOUNTING` only; on the *old* Burner the V3 vote explicitly revokes it from `Lido`, the curated module, SimpleDVT, and CSM accounting — items 1.7–1.10, then `_assertZeroOZRoleHolders(OLD_BURNER, requestBurnSharesRole)`).

**Admin & proxy-admin holders.** `DEFAULT_ADMIN_ROLE` and every `OssifiableProxy` admin = the Aragon **`AGENT`**, governed by DG — DG governs the `AGENT` rather than holding the role directly. `_assertFinalACL` asserts both `_assertSingleOZRoleHolder(..., DEFAULT_ADMIN_ROLE, AGENT)` **and** `_assertProxyAdmin(..., AGENT)` across Burner, VaultHub, OperatorGrid, LazyOracle, AccountingOracle and PredepositGuarantee; `OracleReportSanityChecker` gets the `DEFAULT_ADMIN_ROLE` assertion only, `Accounting` the proxy-admin assertion only, and `StakingRouter` neither (its `REPORT_REWARDS_MINTED_ROLE` = `ACCOUNTING` is asserted instead).

## Core flows

### 1. Dual-governance veto / escrow state machine

DG (cross-repo, `lidofinance/dual-governance`) interposes a timelock + stETH-holder veto between the Aragon DAO and any state-mutating core call. A passed Aragon vote no longer executes immediately: it is wrapped in a `Proposal` (a list of `ExternalCall{target,value,payload}` structs) that the `Executor` performs in one tx — but only after the timelock and state machine permit.

Why it exists: LDO and stETH incentives are misaligned — LDO holders can pass a harmful change; stETH holders can credibly threaten mass exit, cratering LDO. DG turns that threat into an on-chain enforced delay: stETH holders lock tokens in escrow to signal opposition.

```text
stETH/wstETH/unstETH holder ──lock──> VetoSignalingEscrow ──signals──┐
LDO holder ── Aragon Vote ──> DG Timelock (proposal queued) <─────────┘
                                  │  (gated by state machine below)
                                  ▼
                              Executor.execute(proposal)  // EXT: list of ExternalCall
                                  ▼
                              core contracts (Lido, StakingRouter, ...)

Normal ──locked ≥ firstSeal──> VetoSignalling ──locked ≥ secondSeal──> RageQuit
   ▲                                 │ timer expires (< secondSeal)        │ all escrow stETH
   │                                 ▼                                     │ exited via WQ
   └── VetoCooldown <── Deactivation ┘ <─────────────────────────────────┘
       (pending proposals execute, then reset)
```

States and effects:
- **Normal** (locked < `firstSealRageQuitSupport`, ~1% of stETH supply): proposals execute after `DEFAULT_DELAY` (~3 days).
- **VetoSignalling** (locked ≥ first seal): proposal *execution* blocked; new proposals can still queue. Window 5–45 days, scaling with locked %.
- **VetoSignalling Deactivation** (timer expired without crossing second seal): brief transition; new proposals blocked.
- **VetoCooldown** (deactivation completed): pending proposals execute *even if* opposition still > 1% — prevents endless re-vetoing, then resets to Normal.
- **RageQuit** (locked ≥ `secondSealRageQuitSupport`, ~10%): all execution blocked until *every* escrowed token has exited via `WithdrawalQueueERC721`; can last weeks (WQ throughput bound).

**Veto-bypass invariant (the central blast-radius risk).** DG only constrains roles reachable through the Aragon Vote → Agent → DG path. **Any high-impact role held by an address not gated by DG bypasses the veto entirely** — it can be exercised with no timelock and no stETH-holder veto. The role matrix above is exactly the surface to audit for this: a holder column that is *not* "Agent, governed by DG" (e.g. a multisig, an Easy Track factory, or — critically — a compromised `LidoLocator` impl re-routing trust) is a veto-bypass candidate. Treat `firstSeal`/`secondSeal`/`DEFAULT_DELAY` as deployment parameters to confirm against the on-chain DG config, not hardcoded constants.

### 2. Ossification end-state

`OssifiableProxy.proxy__ossify()` is the deliberate, irreversible end-state for a sufficiently-stable component. It is `onlyAdmin`, sets the ERC-1967 admin slot to `address(0)`, and emits `AdminChanged` + `ProxyOssified`.

```text
admin ── proxy__ossify() ──> _ADMIN_SLOT = address(0)
                              emit AdminChanged(prevAdmin, 0); ProxyOssified()
later: proxy__upgradeTo / proxy__upgradeToAndCall / proxy__changeAdmin
       └─> onlyAdmin: admin == 0 ⇒ revert ProxyIsOssified  // permanent
```

After ossification the implementation at that instant is *the* implementation forever; `proxy__getIsOssified()` returns true. Gotcha: ossification freezes the **impl pointer**, not the impl's own internal admin — an ossified locator still has its baked-in immutables fixed, but a non-ossified proxy in front of a critical impl remains the single most dangerous upgrade target (see Internal mechanics). None of the current production proxies are ossified.

## Internal mechanics

**LidoLocator immutable-immutables.** The 22 peer addresses (`lido`, `accountingOracle`, `validatorsExitBusOracle`, `withdrawalQueue`, `withdrawalVault`, `stakingRouter`, `depositSecurityModule`, `oracleReportSanityChecker`, `oracleDaemonConfig`, `burner`, `elRewardsVault`, `accounting`, `vaultHub`, `vaultFactory`, `operatorGrid`, `lazyOracle`, `predepositGuarantee`, `validatorExitDelayVerifier`, `triggerableWithdrawalsGateway`, `wstETH`, `treasury`, `postTokenRebaseReceiver`) are `public immutable`, set once in the constructor and baked into impl bytecode — *not* proxy state. `_assertNonZero` reverts `ZeroAddress` on all except `postTokenRebaseReceiver` (allowed zero). Changing any address = deploy a new impl + repoint the proxy. Hazard: **caller-identity auth is the dominant pattern** (`Lido.deposit` accepts only `LidoLocator.depositSecurityModule()`; report-exec accepts only the accounting orchestrator; `Burner.requestBurnShares` trusts a locator-registered caller). A malicious locator impl can re-route every such check to attacker contracts — hence the locator impl is the single most critical upgrade target, its proxy admin (the Agent) the apex of the blast radius.

**OssifiableProxy admin model.** Admin lives in the ERC-1967 `_ADMIN_SLOT`. `onlyAdmin` reverts `ProxyIsOssified` when admin is zero and `NotAdmin` otherwise — so ossification and "no admin set" are indistinguishable at the call site and both block all four privileged fns. `proxy__getIsOssified()` is literally `_getAdmin() == address(0)`.

**DG escrow mechanics.** `VetoSignalingEscrow` accepts stETH, wstETH (converted to stETH at 1:1, *not* current share rate), and unstETH NFTs (counted at stETH value at request time); tracks per-holder + global locked totals. During Normal/VetoSignalling holders `lockStETH`/`unlockStETH`, subject to a ~5-hour post-lock cooldown that prevents state-manipulating oscillation. On RageQuit the escrow converts locks into WQ requests; DG stays in RageQuit until every request finalizes, then the escrow is replaced fresh and a `RAGE_QUIT_EXTENSION_DELAY` cooldown runs. Gotcha: escalating Veto→RageQuit *commits* you to exiting — no changing your mind once it triggers; the queue must drain.

**GateSeal properties.** One-time use (unusable once activated), per-instance expiry (~1 year; unusable after expiry even if never fired), bounded pause (≤14 days, set at construction), multisig-operated. Each holds `PAUSE_ROLE` on its targets and cannot resume. Max blast radius from a compromised committee: a ≤14-day pause of withdrawals/exits, then auto-resume. **ResealManager** (DG repo) can extend an active GateSeal pause into a governance-controlled indefinite pause — but only while DG is non-Normal. In V3, `_assertFinalACL` asserts both `PAUSE_ROLE` holders (`GateSeal`+`ResealManager`, `_assertTwoOZRoleHolders`) and the sole `RESUME_ROLE` holder (`ResealManager`) on `VaultHub`/`PredepositGuarantee` only; `WithdrawalQueue`'s identical wiring is live deployment config the template does **not** re-assert.

## External interactions

The locator trust-anchor authority map — who is the apex authority over each component, from the in-scope side:

```text
Aragon Vote (LDO) ─wrapped in─> DG proposal ─timelock/veto─> Executor ─> Agent (apex authority)
                                                              ▲ stETH-holder veto via Escrow

Agent (DEFAULT_ADMIN_ROLE + OssifiableProxy admin) over:
  LidoLocator(impl), Burner, Accounting, VaultHub, OperatorGrid, LazyOracle,
  AccountingOracle, VEBO, HashConsensus, OracleReportSanityChecker, StakingRouter,
  WithdrawalQueue(ERC721), PredepositGuarantee, DepositSecurityModule(owner)

Caller-identity auth (resolved through LidoLocator immutables):
  Lido.deposit            <─ accepts only locator.depositSecurityModule()
  Lido report-exec fns    <─ accept only locator.accounting() orchestrator
  Burner.requestBurnShares<─ role + trusts locator-registered caller (Accounting, CSM_ACCOUNTING)

Emergency seams (bypass / short-circuit):
  GateSeal ─PAUSE_ROLE─> WithdrawalQueue, VaultHub, PredepositGuarantee  // 14-day cap, auto-resume
  ResealManager ─extend pause─> (only while DG non-Normal)
  Easy Track factories ─narrow roles─> NodeOperatorsRegistry, OperatorGrid, VaultHub  // 72h objection window, 0.5% LDO veto

External: DG/Escrow/ResealManager/GateSeal/Agent are all out-of-repo authorities.
```

**Easy Track** is optimistic governance for routine ops: anyone enacts a typed-and-bounded motion with a 72-hour objection window during which 0.5% of LDO can veto; unvetoed, it auto-executes. Each motion type is a separate audited factory with a fixed target and bounded payload, holding only its narrow role (e.g. `SET_NODE_OPERATOR_LIMIT_ROLE`, the OperatorGrid factories). Not a generic vote — but per the veto-bypass invariant, every factory's narrow role is a non-DG-timelock authority to audit.

## Key constants

| Constant | Value | Purpose |
|---|---|---|
| `OssifiableProxy` ossified admin | `address(0)` | irreversible end-state; blocks all privileged proxy fns |
| LidoLocator immutables | 22 addresses | peer registry baked into impl bytecode |
| DG `DEFAULT_DELAY` | ~3 days (deployment param) | Normal-state timelock before execute |
| DG `firstSealRageQuitSupport` | ~1% of stETH supply (param) | enters VetoSignalling |
| DG `secondSealRageQuitSupport` | ~10% of stETH supply (param) | enters RageQuit |
| VetoSignalling window | 5–45 days (param) | scales with locked % |
| Escrow lock cooldown | ~5 hours (param) | anti-oscillation after `lockStETH` |
| GateSeal max pause | ≤ 14 days | bounded emergency pause; auto-resume |
| GateSeal expiry | ~1 year (per instance) | one-shot validity window |
| Easy Track objection | 72 hours / 0.5% LDO | optimistic-veto threshold |

DG/Escrow/GateSeal numerics are deployment parameters in external repos — confirm against on-chain config or `dual-governance/docs/specification.md` before relying on exact values.

## Source references

**Live source** (symbols cited above resolve here):
- `0.8.9/LidoLocator.sol` — `Config`, the 22 `public immutable` getters, `_assertNonZero` (`ZeroAddress`), `coreComponents`/`oracleReportComponents`.
- `0.8.9/proxy/OssifiableProxy.sol` — `proxy__ossify`/`proxy__upgradeTo`/`proxy__upgradeToAndCall`/`proxy__changeAdmin`, `onlyAdmin` (`ProxyIsOssified`/`NotAdmin`), `proxy__getIsOssified`, events `ProxyOssified`/`AdminChanged`.
- `upgrade/V3Template.sol` — `_assertFinalACL` (final-ACL truth: `AGENT` as `DEFAULT_ADMIN_ROLE` + proxy admin; `REQUEST_BURN_SHARES_ROLE` = `ACCOUNTING`+`CSM_ACCOUNTING`; `PAUSE_ROLE` = `GATE_SEAL`+`RESEAL_MANAGER`; `RESUME_ROLE` = `RESEAL_MANAGER`; `REPORT_REWARDS_MINTED_ROLE` = `ACCOUNTING`; zero-holder for ORSC limit roles, `VAULT_MASTER_ROLE`, `REDEMPTION_MASTER_ROLE`), `_assertEasyTrackFactoriesAdded`.
- `upgrade/V3VoteScript.sol` — revokes `REQUEST_BURN_SHARES_ROLE` from `Lido`/curated/SimpleDVT/old-CSM-accounting on the old Burner; grants `REPORT_REWARDS_MINTED_ROLE` to `Accounting`, and PDG `PAUSE_ROLE`/config-manager to `AGENT`.
- External authorities: `GateSeal` (Vyper), `lidofinance/dual-governance` (`Timelock`/`Executor`, `Escrow`, `ResealManager`), Aragon `AGENT`/Voting, `lidofinance/easy-track`.

**Official docs (docs/docs/):** `lido-dao.md`, `contracts/lido-locator.md`, `contracts/gate-seal.md`, `contracts/ossifiable-proxy.md`, `guides/dg-guide.md`, `guides/easy-track-guide.md`; external `dual-governance/docs/specification.md` (authoritative state-machine + escrow spec).
