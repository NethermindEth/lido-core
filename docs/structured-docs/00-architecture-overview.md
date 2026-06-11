---
doc: "00"
title: Architecture Overview
contracts: []
prereqs: []
see_also: ["01","03","06"]
ssot_for: [module-map, critical-flows, report-execution-narrative]
---
# 00 — Architecture Overview

> Orientation spine and the **sole SSOT for the report-execution narrative** (critical flow 2). Detail modules link back here for the end-to-end ordering; they own the internal mechanics. Start in [`01`](./01-core-staking.md) (staking/tokens), [`03`](./03-oracle-accounting.md) (report detail), [`06`](./06-vaults.md) (V3 vaults).

## What Lido is

Lido is the largest Ethereum liquid-staking protocol. Users deposit ETH for **stETH**, a rebasing ERC-20 representing a pro-rata share of all protocol-controlled ETH (buffered, in transit, or on the Beacon Chain). The protocol delegates ETH to validators run by approved **staking modules** (Curated, Simple DVT, CSM, and — in V3 — stVaults), tracks validator state via a permissioned **oracle committee**, and redeems stETH through an asynchronous **withdrawal queue** minting an NFT receipt. Every contract resolves peers through **`LidoLocator`** — one immutable service-locator proxy; changing the locator changes the world.

## Modules and layout

```text
                Users / DeFi
          submit ETH |    ^ request exit / claim
                     v    |
01 core-staking   Lido (0.4.24) -- StETH -- wstETH
                     | buffer        ^ rebase (mint/burn shares)
                     v deposit       |
02 staking-router StakingRouter --+-- NodeOperatorsRegistry (Curated / Simple DVT)
                     ^             +-- CSM (cross-repo)
   05 deposit-sec    |          (stVaults: separate V3 path — PDG -> StakingVault -> DepositContract, NOT a StakingRouter module; see 06)
   DSM guardian quorum -> Lido.deposit
03 oracle-acct    Accounting (report orchestrator) -- OracleReportSanityChecker -- Burner
                     ^ handleOracleReport               (oracle ingest: HashConsensus -> AccountingOracle)
04 withdrawals    WithdrawalQueueERC721 -- WithdrawalVault (CL withdrawals) -- TriggerableWithdrawalsGateway (EIP-7002)
08 exits          ValidatorsExitBus(Oracle) -- ValidatorExitDelayVerifier   (VEBO publish + VEDV delay proof)
06 vaults V3      VaultFactory -> StakingVault + Dashboard ; VaultHub ; OperatorGrid ; LazyOracle ; PredepositGuarantee
07 governance     Aragon DAO/ACL (LDO) -- LidoLocator -- OssifiableProxy -- GateSeal ; Dual Governance gate (cross-repo)
R  reference      SSZ / GIndex proofs ; EIP-7002 / 7251 / 4788 distilled specs
```

## The four critical flows

### 1. Submit and deposit (user → validator)

```text
user -> Lido.submit{value}()                     // mint shares @ rate, ETH into buffer
   off-chain depositor bot collects guardian ATTEST quorum
DepositSecurityModule.depositBufferedEther(...)  // verifies deposit root unchanged (LIP-5 front-run guard)
 -> Lido.deposit(maxDeposits, moduleId, ...)
 -> StakingRouter.deposit()                       // allocate via MinFirstAllocationStrategy (least-used module first)
 -> IStakingModule.obtainDepositData()            // keys from chosen module
 -> BeaconChainDepositor -> IDepositContract      // EXT: 32 ETH per validator key
```
`submit` mints shares at the current rate into **`bufferedEther`** (no validator push). A guardian quorum signs an `ATTEST` over the deposit root, DSM re-checks it is unchanged since signing (LIP-5 front-run guard), then ETH flows through `StakingRouter`. Detail: [`01`](./01-core-staking.md), [`05`](./05-deposit-security.md), [`02`](./02-staking-router-modules.md).

### 2. Oracle report execution (validator state → stETH supply) — CANONICAL NARRATIVE

The protocol's heartbeat and most security-sensitive flow. Narrated end-to-end here once; [`03`](./03-oracle-accounting.md) owns per-step mechanics, [`04`](./04-withdrawals.md#core-flows) the finalize chain.

```text
oracle members -> HashConsensus.submitReport(hash)      // accounting frame ~225 epochs (~24h), anchored ~12:00 UTC
 quorum (e.g. 5/9) -> AccountingOracle.submitReportData(ReportData)
AccountingOracle -> StakingRouter.updateExitedValidatorsCountByStakingModule(...)
AccountingOracle -> WithdrawalQueue.onOracleReport(bunkerMode, ...)   // relays bunker state ONLY, no finalize
AccountingOracle -> Accounting.handleOracleReport(ReportValues)
  Accounting._applyOracleReportContext, IN ORDER:
   1. _sanityChecks via OracleReportSanityChecker        // bound rebase magnitude + exited churn
   2. IF sharesToFinalizeWQ>0: Burner.requestBurnShares(withdrawalQueue, sharesToFinalizeWQ)  // QUEUE WQ shares
   3. Lido.processClStateUpdate(...)                      // commit new CL balance + validator counts
   4. IF badDebt>0: VaultHub.decreaseInternalizedBadDebt -> Lido.internalizeExternalBadDebt   // socialize vault loss
   5. IF totalSharesToBurn>0: Burner.commitSharesToBurn(total) -> Lido.burnShares   // commit AGGREGATE (WQ+cover/non-cover)
   6. Lido.collectRewardsAndProcessWithdrawals(...)
        -> ELRewardsVault.withdrawRewards() ; WithdrawalVault.withdrawWithdrawals()  // EXT: pull into buffer
        -> WithdrawalQueue.finalize{value}(...)   // lock ETH for batch @ checkpoint rate
   7. IF sharesToMintAsFees>0: Lido.mintShares(address(this)=Accounting) -> _distributeFee (transferShares -> modules + treasury) -> StakingRouter.reportRewardsMinted  // FEES LAST
   8. Lido.emitTokenRebase(...)                           // post-rebase event
External: HashConsensus, AccountingOracle, ELRewardsVault, WithdrawalVault, EIP-4788 (proofs, separate path).
```
**Why the ordering matters.** WQ-finalized shares are *queued* by `Burner.requestBurnShares` (step 2) but only *committed in aggregate* by `Burner.commitSharesToBurn(total)` (step 5), which drives `Lido.burnShares` — do not conflate the two. Burns commit **before** fees mint, so the burn happens at the pre-mint rate; **fees mint LAST** (step 7), settling against the already-rebased rate and never diluting the burn. Bad debt from insolvent V3 vaults is internalized (step 4) into the core share base before the burn/finalize, socializing the loss across all stETH holders. The accounting share rate is **internal ether / internal shares** (external vault ether/shares excluded; internal-ether composition in [`01`](./01-core-staking.md#internal-mechanics)). A **second, independent oracle** (separate `HashConsensus` + `ValidatorsExitBusOracle`/VEBO on a shorter frame, separate committee) publishes validator-exit requests (→ [`08`](./08-exits.md#core-flows)); it does not touch this accounting path. Detail: [`03`](./03-oracle-accounting.md).

### 3. Withdrawal (stETH → ETH)

```text
user -> WithdrawalQueueERC721.requestWithdrawals(amounts, owner)   // lock stETH/wstETH, mint unstETH NFT
   ... an oracle report (flow 2) finalizes a contiguous batch:
       prefinalize -> requestBurnShares -> commitSharesToBurn -> collectRewardsAndProcessWithdrawals -> WithdrawalQueue.finalize ...
user -> WithdrawalQueueERC721.claimWithdrawal(requestId)           // pay ETH @ finalization rate
   ETH source: WithdrawalVault (CL withdrawals via 0x01/0x02 creds, pulled in flow 2)
```
Requests are not paid in id order on demand; each report advances `lastFinalizedRequestId` over a **contiguous** batch (≤ `MAX_BATCHES_LENGTH = 36` per finalize), locking a precise ETH amount at a checkpointed share rate. A holder claims only once the request id is finalized. The on-report finalize chain and the bunker-mode relay (`onOracleReport` never finalizes) are detailed in [`04`](./04-withdrawals.md#core-flows).

### 4. Vault mint and burn (V3 stVaults)

```text
staker -> VaultFactory.createVaultWithDashboard()    // deploy StakingVault + Dashboard, connect to VaultHub
staker -> Dashboard.fund{value} -> StakingVault       // ETH into vault
staker -> Dashboard.mintShares -> VaultHub.mintShares -> Lido.mintExternalShares   // mint stETH vs overcollateral
PredepositGuarantee proves each validator WC (EIP-4788) -> StakingVault.depositToBeaconChain -> DepositContract
LazyOracle -> VaultHub.applyVaultReport               // NAV via Merkle root + quarantine for sudden jumps
   ... burn / rebalance / force-exit / disconnect ...
   VaultHub.burnShares -> Lido.burnExternalShares     // repay liability
```
A V3 stVault is a separate non-custodial contract whose withdrawal credentials the staker owns; minting is bounded by the `OperatorGrid` tier (reserve ratio, share limit, fees), and below the health threshold the vault is force-rebalanced or its shortfall socialized as bad debt (internalized in flow 2). Vault NAV/fees flow on a separate `LazyOracle` path, **not** the accounting report; force-exit uses `TriggerableWithdrawals` directly, not the gateway. Detail: [`06`](./06-vaults.md).

## Governance

Three layers. (1) **Aragon DAO** (LDO) — root authority; on-chain votes set roles and upgrade contracts. The 0.4.24 contracts (`Lido`, `StETH`, `NodeOperatorsRegistry`) use the Aragon ACL with `bytes32` role constants; the 0.8.x contracts use OZ `AccessControl`, where `DEFAULT_ADMIN_ROLE` (and OZ-core admin) resolves to the Aragon `AGENT`, governed by Dual Governance. (2) **Easy Track** — DAO-pre-approved motion framework for bounded recurring ops; out of core scope except role assignments. (3) **Dual Governance** (cross-repo) — a timelock + escrow gate wrapping Aragon votes affecting the core; stakers lock stETH/wstETH/unstETH into escrow to enter Veto Signaling, and at the second-seal threshold the protocol enters Rage Quit, freezing upgrades until lockers exit. Full role matrix: [`07`](./07-governance-permissions.md).

## Emergency response

- **`GateSeal`** — single-use one-of-N committee that pauses `WithdrawalQueue`/`ValidatorsExitBusOracle` for a bounded window (cap set at construction, ≤ 14 days), then expires. Circuit breaker for mid-frame bugs.
- **`pauseStaking`** (`STAKING_PAUSE_ROLE`) / **`stop`** (`PAUSE_ROLE`) on `Lido` — halt submissions / routine ops.
- **`pauseDeposits`** on `DepositSecurityModule` — any single guardian with a fresh signed `PAUSE_MESSAGE` stops deposits during a suspected front-run; owner resumes via `unpauseDeposits()`.

## Source references

**Live source:** `0.8.9/Accounting.sol` (`_applyOracleReportContext` report ordering), `0.4.24/Lido.sol` (`submit`/`_getInternalEther`/`_getShareRate*`/`collectRewardsAndProcessWithdrawals`), `0.8.9/Burner.sol` (`requestBurnShares`/`commitSharesToBurn`), `0.8.9/LidoLocator.sol`.

Official docs: `docs/introduction.mdx`, `docs/lido-v3-whitepaper.mdx`, `docs/contracts/lido-locator.md`, `docs/guides/dg-guide.md`.
