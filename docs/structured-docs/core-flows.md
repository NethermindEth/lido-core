  ---
  doc: "core-flows"
  title: Critical Flows — Supplementary Context
  contracts: []
  prereqs: []
  see_also: ["00", "01", "02", "03", "04", "05"]
  ssot_for: []
  ---
  # Critical Flows — Supplementary Context (Core Pool)

  > Four flows in the Core Pool that touch multiple contracts and are worth orienting around early. This file is **supplementary context, not a scope boundary** — it highlights important paths, but the agent should also examine flows, code paths, and edge cases not covered here. Use this as one lens among many.

  ## What's in this file
  - Flow A — Oracle report and token rebase.
  - Flow B — Deposit allocation & validator accounting.
  - Flow C — Withdrawal finalization and claim.
  - Flow D — EIP-7002 triggerable validator exits.

  Each entry has: *Why it matters*, *What it does* (numbered steps), *Contracts involved*.

  ## What this file is NOT
  - Not the canonical mechanics for any flow. Per-step mechanics live in the per-contract structured docs.
  - Not exhaustive. Smaller flows (administrative operations, view-only paths, edge-case branches) are out of this file's frame and still in scope for review.
  - Not an ordering signal. The "Flow A/B/C/D" naming is for reference inside this file only; it does not imply that other paths deserve less scrutiny.

  ## How to use it
  Read this first to get oriented on the multi-contract flows. Then go to the per-contract docs for mechanics, and to the source for everything else.

## **Flow A: Oracle report and rebase flow**

### Why it matters
This is the protocol's repricing event, the only transaction that can change the share-to-ETH exchange rate, and the change lands on every stETH holder simultaneously. The `OracleReportSanityChecker` is the only on-chain bound on the damage a single report can cause, so the safety of this flow is the safety of the entire protocol's accounting. Every other flow reads from the state this one writes.

### What it does
The single transaction (initiated by `AccountingOracle.submitReportData`) that re-prices every stETH share. In one call:

1. Mutates the share-to-ETH exchange rate: changes every stETH holder's balance.
2. Mints new shares as protocol/NO/treasury fees (dilution of all holders).
3. Burns shares (cover + non-cover) : absorbs losses.
4. Finalizes withdrawal batches : locks in the share rate each batch claims at, and locks ETH for claims.
5. Pushes CL state, EL rewards, and WithdrawalVault ETH into Lido's books.
6. Internalizes bad debt from external (vault) shares: dilutes the internal share rate.

### Contracts involved
- `AccountingOracle.sol`: entry, consensus binding
- `Accounting.sol` : orchestrator (10-step body of `_applyOracleReportContext`)
- `OracleReportSanityChecker.sol` : bounds (`smoothenTokenRebase`, simulated-rate check, CL-decrease check)
- `Lido.sol` / `StETH.sol` : share / ether state writes
- `Burner.sol` : cover / non-cover burn commit
- `WithdrawalQueueBase.sol` : `prefinalize` + `_finalize`
- `StakingRouter.sol` : fee distribution + `reportRewardsMinted`


## Flow B: Deposit Allocation & Validator Accounting 

### Why it matters
This is how staked ETH actually leaves the protocol for the consensus layer. Each 32-ETH deposit is irreversible until the validator exits, so allocation mistakes such as depositing to unvetted keys, mis-counting active validators, mis-routing reward shares,  materialize as permanent skew rather than recoverable losses. The integrity of every subsequent oracle report (Flow A) depends on the validator counts and module bookkeeping written here.

### What it does
A user sends ETH to Lido and receives stETH shares. The buffered ETH eventually gets allocated to staking modules and deposited to the beacon chain.

1. `Lido.submit()` / `receive()`: pulls ETH, mints shares at the *pre-buffer-increase* rate, adds to buffer.
2. Periodically, `Lido.deposit()` pulls ETH from the buffer and forwards to `StakingRouter.deposit()`.
3. StakingRouter allocates deposits across modules via `MinFirstAllocationStrategy` (min-first water-filling, respecting per-module `stakeShareLimit`).
4. The chosen module's `obtainDepositData()` returns signing keys, `BeaconChainDepositor` sends 32 ETH per validator.

### Contracts touched
- `Lido.sol`: submit entry, buffer accounting
- `StakingRouter.sol`: module registry, allocation, deposit dispatch
- `MinFirstAllocationStrategy.sol`: allocation algorithm
- `NodeOperatorsRegistry.sol`: curated module supplying signing keys
- `BeaconChainDepositor.sol`:  beacon-chain deposit forwarder


## Flow C: Withdrawal Finalization and Claim 

### Why it matters
This is the only path ETH leaves the protocol back to users. Both ends are permissionless, anyone can request a withdrawal, anyone can claim one, and the checkpoint / discount math is the only thing standing between a bug and a direct drain. Bunker mode is the protocol's switch for socializing losses fairly across exiting and remaining holders, bypassing it lets exiting users dodge their share of any loss.

### What it does
A user converts stETH back to ETH via a three-stage NFT-mediated process spanning at least one oracle report:

1. `WithdrawalQueue.requestWithdrawals()`: pulls stETH, enqueues request, mints an NFT.
2. Oracle picks finalization batches off-chain (`calculateFinalizationBatches`), submits them in `withdrawalFinalizationBatches`.
3. Accounting calls `prefinalize` (pricing) then `_finalize` (locks ETH, burns shares via Burner, writes one checkpoint per finalization).
4. User calls `WithdrawalQueue.claim()`: the checkpoint determines payout (nominal or discounted).

### Contracts touched
- `WithdrawalQueue.sol` + `WithdrawalQueueBase.sol`: request / finalize / claim, checkpoint accounting
- `WithdrawalQueueERC721.sol` : NFT representation
- `Burner.sol`: receives finalization shares to burn
- `Accounting.sol`: bridges WQ into the rebase
- `WithdrawalVault.sol`: CL withdrawal sink

## Flow D: EIP-7002 Triggerable Validator Exits 

### Why it matters 
This is how the protocol enforces validator exits, for unresponsive or misbehaving operators, for slashed validators, or for protocol-level rebalancing. Without it the protocol cannot reclaim stake on its own schedule. The flow is permissioned (role-gated entry, gateway-only vault auth), so its risk profile is griefing and seam-crossing authorization rather than open value extraction.

### What it does
A permissioned caller (typically `ValidatorsExitBus` or DAO ops) forces specific validators to exit by submitting EIP-7002 withdrawal requests through the WithdrawalVault.

1. Caller invokes `TriggerableWithdrawalsGateway.triggerFullWithdrawals(validators, refundRecipient, exitType)`.
2. Gateway consumes rate-limit budget (`ExitLimitUtils`), checks fee, forwards to `WithdrawalVault.addWithdrawalRequests`.
3. WithdrawalVault calls the EIP-7002 precompile per validator (sending the per-request fee).
4. Excess `msg.value` is refunded to `refundRecipient`.
5. Gateway notifies StakingRouter via `onValidatorExitTriggered` so modules track exit reasons.

### Contracts touched
- `TriggerableWithdrawalsGateway.sol`: entry, rate limit, refund
- `WithdrawalVault.sol` + `WithdrawalVaultEIP7002.sol`: EIP-7002 precompile call
- `ExitLimitUtils.sol`: packed rate-limit state
- `StakingRouter.sol`:  `onValidatorExitTriggered` notification
