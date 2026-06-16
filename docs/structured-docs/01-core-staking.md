---
doc: "01"
title: Core Staking
contracts: [Lido, StETH, StakeLimitUtils]
prereqs: []
see_also: ["00","03","07"]
ssot_for: [share-rate, external-shares, stake-limit, totalPooledEther]
---
# 01 — Core Staking

> The deposit entry point and the **stETH share-accounting root**: every other module's share/ether math resolves against `Lido` here. Deposits leave for validators via [`05` deposit-security](./05-deposit-security.md); the rebase that moves the share rate is driven by [`03` accounting](./03-oracle-accounting.md); the external-shares seam serves the V3 vaults in [`06`](./06-vaults.md). Roles are indexed in [`07`](./07-governance-permissions.md).

## Contracts

| Contract | File | Role |
|---|---|---|
| `Lido` | `0.4.24/Lido.sol` | Liquid-staking pool + rebasing stETH; submit/buffer, deposit gate, report-driven mutators, external-shares seam, protocol levers. Aragon app; inherits `StETH`. |
| `StETH` | `0.4.24/StETH.sol` | Abstract shares accounting + ERC-20 surface; defines share-rate via overridable numerator/denominator hooks that `Lido` implements. |
| `StakeLimitUtils` | `0.4.24/lib/StakeLimitUtils.sol` | Packed single-slot stake-limit state + regen-accumulator helpers; `using` library in `Lido`. |
| `StakingRouter` (boundary) | `0.8.9/StakingRouter.sol` | Receives `deposit{value}` and places 32-ETH chunks to a module. Detail in [`02`](./02-staking-router-modules.md). |
| `Accounting` (boundary) | `0.8.9/Accounting.sol` | Drives the rebase by calling Lido's report mutators. Detail in [`03`](./03-oracle-accounting.md). |
| `VaultHub` (boundary) | `0.8.25/vaults/VaultHub.sol` | Sole caller of the external-shares mint/burn/rebalance seam. Detail in [`06`](./06-vaults.md). |

## Core flows

### 1. Submit (deposit) — LOAD-BEARING

The user-facing entry. The default payable fallback routes to `_submit(address(0))`, so a plain ETH transfer with empty calldata is treated as a stake (this is why EL/withdrawal intake uses *dedicated* hooks — flow 4). Ordering matters: the stake-limit accumulator is decremented (and the pause check runs) *before* shares are computed, so a paused or exhausted limit reverts before any state moves.

```text
user → Lido.submit(referral)  (payable)  // or fallback receive() == submit(0)
  → require(msg.value != 0)                            // ZERO_DEPOSIT
  → _decreaseStakingLimit(msg.value)                   // reverts STAKING_PAUSED / STAKE_LIMIT; consumes accumulator
  → shares = getSharesByPooledEth(msg.value)           // = msg.value * internalShares / internalEther (round down)
  → _mintShares(msg.sender, shares)                    // totalShares += shares; recipient != 0 and != stETH
  → _setBufferedEther(bufferedEther + msg.value)        // buffer rises AFTER mint, but totalPooledEther unchanged net
  → emit Submitted; emit Transfer(0, sender, value)    // via _emitTransferAfterMintingShares
External: none (permissionless).
```

Net invariant: a submit adds `msg.value` to internal ether (buffer) and matching shares, so the share rate is unchanged. On the first deposit the divisor is the `0xdead` initial-holder shares (`_mintInitialShares`, "stone in the elevator") — `internalShares` is never 0, so `getSharesByPooledEth` cannot divide by zero.

### 2. Deposit to validators

```text
DSM → Lido.deposit(maxDepositsCount, moduleId, depositCalldata)
  → require(msg.sender == LidoLocator.depositSecurityModule())   // APP_AUTH_DSM_FAILED
  → require(canDeposit())                                         // !bunkerMode && !isStopped — CAN_NOT_DEPOSIT
  → count = min(maxDepositsCount, StakingRouter.getStakingModuleMaxDepositsCount(moduleId, getDepositableEther()))
  → // CEI: update state BEFORE the external call to block reentrancy
  → _setBufferedEtherAndDepositedValidators(buffered - count*32, depositedValidators + count)
  → emit Unbuffered; emit DepositedValidatorsChanged
  → StakingRouter.deposit{value: count*32}(count, moduleId, calldata)   // EXT: must place ALL ether or revert whole tx
External: StakingRouter (→ staking module → beacon deposit contract).
```

`getDepositableEther` = `bufferedEther − withdrawalQueue.unfinalizedStETH()` (clamped at 0): buffer earmarked for unfinalized withdrawals is never deposited. The buffer drop is exactly offset by a transient-ether rise (new `depositedValidators` not yet seen on CL), so `totalPooledEther` is unchanged. If the router cannot place all ether it MUST revert — partial placement would desync `bufferedEther` from on-chain reality.

### 3. Share rate and rebase math — LOAD-BEARING

stETH has no stored balances: `balanceOf(a) = sharesOf(a) * numerator / denominator`. `Lido` **overrides** the StETH base hooks (which default to total/total) so the rate uses INTERNAL totals only:

```text
shareRate = _getShareRateNumerator() / _getShareRateDenominator()
          = _getInternalEther()      / (totalShares - externalShares)
_getInternalEther() = bufferedEther + transientEther + clBalance
  transientEther = (depositedValidators - clValidators) * 32 ETH      // in-flight deposits
```

There is **no `1e27` factor on the token** — the `1e27`-scaled `preShareRate`/`postShareRate` in the `TokenRebased` event comments are the oracle/WQ APR convention, not the token's conversion. External shares are *excluded* from the rate so minting/burning vault-backed shares cannot dilute the rate for ordinary holders; `totalPooledEther = internalEther + externalEther` with `externalEther = externalShares * internalEther / internalShares` keeps both share types at the same price. There is **no `Transfer` event on rebase** — only on explicit transfers; integrators watch `TokenRebased`. The rebase is not one entrypoint: `Accounting` (flow 5) calls a sequence of mutators that move CL balance, buffer, and shares, each shifting the rate.

### 4. EL rewards and withdrawal intake

```text
LidoExecutionLayerRewardsVault → Lido.receiveELRewards()  (payable)
  → _auth(_elRewardsVault())                              // locator-resolved caller
  → TOTAL_EL_REWARDS_COLLECTED += msg.value               // buffer NOT bumped here
  → emit ELRewardsReceived

WithdrawalVault → Lido.receiveWithdrawals()  (payable)
  → _auth(_withdrawalVault())
  → emit WithdrawalsReceived                              // only an event; no buffer write
External: locator-registered EL-rewards vault / withdrawal vault only.
```

Crucial subtlety: neither hook touches `bufferedEther`. The ETH sits on the contract balance; the buffer figure only rises later, inside `collectRewardsAndProcessWithdrawals`, when `Accounting` pulls these amounts during the report (flow 5). This separation is why a stray ETH send with empty calldata mints user shares (fallback == submit) rather than silently becoming protocol rewards.

### 5. Report-driven mutators (← Accounting / Burner)

`Lido` has no `handleOracleReport`; that lives on `Accounting`, which calls these Lido mutators in order, each locator-gated (`_auth(_accounting)`) and — except the event-only `emitTokenRebase` — `_whenNotStopped`.

```text
Accounting.handleOracleReport   (detail in 03)
  → Lido.processClStateUpdate(...)                       // _auth(_accounting); set CL balance/validators
  → Lido.internalizeExternalBadDebt(shares)              // _auth(_accounting); vault bad-debt seam (flow 6) — runs BEFORE collect/mint
  → Lido.collectRewardsAndProcessWithdrawals(...)        // _auth(_accounting); the settlement step:
        if elRewards > 0:   ELRewardsVault.withdrawRewards(elRewards)        // EXT: pull
        if withdrawals > 0: WithdrawalVault.withdrawWithdrawals(withdrawals) // EXT: pull
        if etherToLockWQ>0: WithdrawalQueue.finalize{value}(lastReqId, rate) // EXT: send + assign burn shares
        bufferedEther = bufferedEther + elRewards + withdrawals - etherToLockWQ   // buffer settled HERE
        emit ETHDistributed
  → Lido.mintShares(Accounting, feeShares)               // _auth(_accounting); fees minted to Accounting, then transferShares → modules + treasury (02)
  → Lido.emitTokenRebase(...)                            // _auth(_accounting); TokenRebased + InternalShareRateUpdated
Burner.commitSharesToBurn → Lido.burnShares(shares)      // _auth(_burner); burn cover/WQ-finalized shares
External: Accounting, EL/withdrawal vaults, WithdrawalQueue, Burner.
```

`mintShares`/`burnShares` emit `Transfer(0,…)` / `SharesBurnt` (Lido historically never emits `Transfer` to/from zero for these — it uses `SharesBurnt`). The WQ-finalize and burn-chain granularity is owned by [`04`](./04-withdrawals.md#core-flows).

### 6. External-shares seam (← VaultHub) — LOAD-BEARING

The V3 stVault seam: VaultHub mints stETH against vault collateral that lives *outside* Lido's buffer/CL. All three are `_auth(_vaultHub())` + `_whenNotStopped`.

```text
VaultHub → Lido.mintExternalShares(recipient, shares)
  → require(shares != 0); require(shares <= _getMaxMintableExternalShares())   // EXTERNAL_BALANCE_LIMIT_EXCEEDED
  → _decreaseStakingLimit(getPooledEthByShares(shares))   // mint consumes stake-limit like a submit
  → externalShares += shares; _mintShares(recipient, shares)   // recipient != stETH contract
VaultHub → Lido.burnExternalShares(shares)                // allowed while staking PAUSED, blocked while STOPPED
  → externalShares -= shares; _burnShares(msg.sender, shares)
  → if limit set and not paused: prevStakeLimit = currentLimit + stethAmount   // unbounded — may exceed maxStakeLimit and drain per-block; does NOT add ETH to buffer
VaultHub → Lido.rebalanceExternalEtherToInternal(shares)  (payable)
  → require(msg.value == getPooledEthBySharesRoundUp(shares))   // VALUE_SHARES_MISMATCH
  → externalShares -= shares; bufferedEther += msg.value         // 1:1 pay-down of vault debt in ETH
Accounting → Lido.internalizeExternalBadDebt(shares)      // (flow 5) NOT VaultHub-direct
  → externalShares -= shares                                     // totalShares same, internalEther same
External: VaultHub (mint/burn/rebalance); Accounting (bad-debt).
```

`_getMaxMintableExternalShares` enforces `(externalShares + x)/(totalShares + x) <= maxExternalRatioBP/10000`, solved as `x = (totalShares*maxBP − externalShares*10000)/(10000 − maxBP)`; returns 0 if `maxBP == 0` or already over, `2^256−1` if `maxBP == 10000`. **Bad-debt internalize** is the only path that socializes a single vault's loss: dropping `externalShares` while keeping `totalShares`/`internalEther` constant cuts `externalEther`, so `totalPooledEther` falls and the share rate drops for *all* holders. Distinct from `rebalanceExternalEtherToInternal`, which pays real ETH into the buffer 1:1 against the burned external shares (only sub-wei rounding, since `msg.value` matches `getPooledEthBySharesRoundUp`) — so vault debt pay-down does not socialize a loss.

### 7. Protocol levers and transfers

```text
PAUSE_ROLE → stop()                  // global freeze; also pauses staking; gates _whenNotStopped mutators
RESUME_ROLE → resume()               // reverse; re-applies prior stake limit
STAKING_PAUSE_ROLE → pauseStaking()  // blocks new submit only; pool ops continue
STAKING_CONTROL_ROLE → resumeStaking() / setStakingLimit(max, perBlock) / removeStakingLimit() / setMaxExternalRatioBP(bp)
UNSAFE_CHANGE_DEPOSITED_VALIDATORS_ROLE → unsafeChangeDepositedValidators(n)   // patch counter when onboarding rotated-WC validators
holder → transfer / transferFrom / transferShares / transferSharesFrom   // _whenNotStopped; recipient != 0 and != stETH
holder → approve / increaseAllowance / decreaseAllowance / permit         // NOT stop-guarded
```

`transfer*` move stETH amounts (converted to shares); `transferShares*` move raw shares — all under `_whenNotStopped`, so `stop()` freezes transfers but allowance setters and `permit` (EIP-2612) still work. Roles are indexed in [`07`](./07-governance-permissions.md).

## Internal mechanics

**StakeLimitUtils — packed single-slot accumulator.** `STAKING_STATE_POSITION` packs four fields into one 256-bit slot: `maxStakeLimit` (uint96, bits 160-255), `maxStakeLimitGrowthBlocks` (uint32, 128-159), `prevStakeLimit` (uint96, 32-127), `prevStakeBlockNumber` (uint32, 0-31). Sentinels: **paused** ⇔ `prevStakeBlockNumber == 0`; **unlimited** ⇔ `maxStakeLimit == 0` (with `prevStakeBlockNumber != 0`). `calculateCurrentStakeLimit` branches on `prevStakeLimit < maxStakeLimit`. With `change = blocksPassed * (maxStakeLimit / maxStakeLimitGrowthBlocks)`: if true, returns `min(prevStakeLimit + change, maxStakeLimit)` (refill, capped); else `max(_saturatingSub(prevStakeLimit, change), maxStakeLimit)` (drain, floored — `prevStakeLimit > maxStakeLimit` is reachable via `burnExternalShares`, decays per-block). Branchless via `_constGasMin/_constGasMax/_saturatingSub` (gas independent of block delta). Each `submit`/`mintExternalShares` calls `_decreaseStakingLimit`: reverts `STAKING_PAUSED`, reverts `STAKE_LIMIT` if `amount > currentLimit`, else writes `prevStakeLimit = currentLimit − amount` and stamps `prevStakeBlockNumber = block.number`. `burnExternalShares` does the inverse without moving ETH. The accumulator is in **wei** — **no `1e3 wei` precision constant**.

**Hard caps.** `setStakingLimit` requires `maxStakeLimit <= uint96.max / 2` (Lido-level) and the library additionally requires `maxStakeLimit <= uint96.max`, `maxStakeLimit >= perBlock`, and `maxStakeLimit/perBlock <= uint32.max`. `setStakingLimit` resets `prevStakeLimit` to the new max only when staking was paused/unlimited or the new max is below the old `prevStakeLimit`.

**Recipient restriction.** `_mintShares`/`_transferShares` reject `address(0)` and `address(this)` (the stETH contract). Minting to the token itself reverts (`MINT_TO_STETH_CONTRACT`), so the external-shares and fee-mint paths inherit that guard.

**Conversion rounding.** `getSharesByPooledEth` rounds **down** (`eth * denom / numer`), `getPooledEthByShares` rounds **down**, `getPooledEthBySharesRoundUp` uses `ceilDiv` — used by `rebalanceExternalEtherToInternal` so the vault never underpays. Inputs must be `< UINT128_MAX`. Total shares are stored in the low 128 bits with a `SHARES_OVERFLOW` mask check on mint.

**Stuck-validators touchpoint.** `stuckValidatorsCount` is DEPRECATED protocol-wide; at the staking side the count is hardcoded to 0 and oracle extra-data `itemType=1` reverts `DeprecatedExtraDataType`. Full detail in [`02`](./02-staking-router-modules.md).

**Contract version (mainnet).** `Lido` is currently at version 3 — `initialize` sets v3, and the v2→v3 path `finalizeUpgrade_v3` (already executed) also wired the initial `maxExternalRatioBP` that flow 7's `setMaxExternalRatioBP` adjusts at runtime.

## External interactions

```text
INBOUND  DSM          → deposit                                           (msg.sender gate)
INBOUND  Accounting   → processClStateUpdate / internalizeExternalBadDebt / collectRewardsAndProcessWithdrawals
                         / mintShares / emitTokenRebase   (_auth(_accounting))
INBOUND  Burner       → burnShares                                        (_auth(_burner))
INBOUND  VaultHub     → mintExternalShares / burnExternalShares / rebalanceExternalEtherToInternal  (_auth(_vaultHub))
INBOUND  EL/WV vaults → receiveELRewards / receiveWithdrawals             (locator-gated)
OUTBOUND Lido → StakingRouter.deposit{value}                              (validator placement)
OUTBOUND Lido → ELRewardsVault.withdrawRewards / WithdrawalVault.withdrawWithdrawals  (report pull)
OUTBOUND Lido → WithdrawalQueue.finalize{value}                          (report finalize)
LOOKUP   Lido → LidoLocator                                              (resolves every gated caller/callee)
EVENTS   Submitted, Unbuffered, DepositedValidatorsChanged, ELRewardsReceived, WithdrawalsReceived,
         ETHDistributed, TokenRebased, InternalShareRateUpdated, ExternalSharesMinted/Burnt,
         ExternalBadDebtInternalized, Transfer, TransferShares, SharesBurnt
```

> **Invariants:** see [`core-invariants.md` §1](./core-invariants.md#1-core-staking-and-tokens) — supplementary, not the full set; derive others from source.

## Key constants

| Constant | Value | Purpose |
|---|---|---|
| `DEPOSIT_SIZE` | 32 ether | Beacon deposit chunk; also the transient-ether unit |
| `TOTAL_BASIS_POINTS` | 10000 | BP denominator (external-ratio cap, fees) |
| `INITIAL_TOKEN_HOLDER` | `0xdead` | Holds bootstrap shares so internal shares are never 0 |
| `UINT128_MAX` | `2^128 − 1` | Upper bound on conversion inputs and total shares |
| stake-limit max cap | `uint96.max / 2` | `setStakingLimit` ceiling on `maxStakeLimit` |
| stake-limit unit | 1 wei | Accumulator denomination (there is NO `1e3 wei` constant) |
| `maxExternalRatioBP` | governance-set | Cap on external/total shares ratio |

## Source references

**Live source** (every symbol cited inline above resolves against these files):
- `0.4.24/Lido.sol` — submit/deposit, share mint/burn, external-shares seam (`mintExternalShares`/`burnExternalShares`/`rebalanceExternalEtherToInternal`/`internalizeExternalBadDebt`), report mutators (`processClStateUpdate`/`collectRewardsAndProcessWithdrawals`/`emitTokenRebase`), `receiveELRewards`/`receiveWithdrawals`, `_getInternalEther`/`_getShareRate*`/`_getMaxMintableExternalShares`, `_decreaseStakingLimit`, roles.
- `0.4.24/StETH.sol` — `getSharesByPooledEth`/`getPooledEthByShares[RoundUp]`, `_transfer*`/`_mintShares`/`_burnShares`/`_mintInitialShares`, share-rate hooks, `INITIAL_TOKEN_HOLDER`/`UINT128_MAX`.
- `0.4.24/lib/StakeLimitUtils.sol` — `StakeLimitState.Data` packed slot, `calculateCurrentStakeLimit`/`setStakingLimit`/`isStakingPaused`/`isStakingLimitSet`, branchless `_constGasMin`/`_constGasMax`/`_saturatingSub`.

**Official docs (`docs/docs/`):** `contracts/lido.md`, `guides/lido-tokens-integration-guide.md` (rebase-event caveats), `guides/protocol-levers.md`.
