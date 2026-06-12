## Deployment state

Current on-chain versions of upgradeable contracts (mainnet):

| Contract | Current version | Last upgrade step |
|---|---|---|
| `Lido` | v3 | `finalizeUpgrade_v3(_oldBurner, _contractsWithBurnerAllowances, _initialMaxExternalRatioBP)` already executed |
| `Burner` | v1 | Deployed at v1; `migrate(_oldBurner)` already executed; `isMigrationAllowed = false` |
| `NodeOperatorsRegistry` | v4 | `finalizeUpgrade_v4` already executed |
| `StakingRouter` | v3 | `finalizeUpgrade_v3` already executed |
| `WithdrawalQueueERC721` | v1 | Initialized; `initialize(...)` already executed |
| `WithdrawalVault` | v2 | `finalizeUpgrade_v2` already executed |

### Upgrade pattern in use

All contracts above are Aragon-style upgradeable and share the same initializer pattern:

```solidity
function finalizeUpgrade_vN(...) external {
    _checkContractVersion(N - 1);
    _setContractVersion(N);
    // ... migration work ...
}

The pair _checkContractVersion(N-1) + _setContractVersion(N) is the access-control mechanism. No onlyRole(...) modifier is used on these functions; the pattern is consistent across every Aragon/Lido upgradeable contract listed. Burner.migrate(...) follows the same one-shot idea but uses an isMigrationAllowed flag instead of a version counter.