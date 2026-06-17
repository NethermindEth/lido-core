## Deployment state

The V3 stVaults contracts are **already deployed and initialized on mainnet** (recorded in [`deployed-mainnet.json`](../deployed-mainnet.json); listed at [`docs.lido.fi/deployed-contracts/`](https://docs.lido.fi/deployed-contracts/)):

| Contract | Proxy / type | Initialization (already executed) |
|---|---|---|
| `VaultHub` | `OssifiableProxy` (singleton) | `initialize(admin)` run in the proxy constructor |
| `LazyOracle` | `OssifiableProxy` (singleton) | `initialize(...)` run in the proxy constructor |
| `OperatorGrid` | `OssifiableProxy` (singleton) | `initialize(...)` run in the proxy constructor |
| `PredepositGuarantee` | `OssifiableProxy` (singleton) | `initialize(...)` run in the proxy constructor |
| `VaultFactory` | non-upgradeable | constructor (immutables set; no initializer) |
| `StakingVault` | `PinnedBeaconProxy` (per vault, beacon) | `initialize(...)` run in the same tx as deploy, by `VaultFactory` |
| `Dashboard` | `Clones` proxy (per vault) | `initialize(...)` run in the same tx as deploy, by `VaultFactory` |

### Upgrade pattern in use

The vault singletons (`VaultHub`, `LazyOracle`, `OperatorGrid`, `PredepositGuarantee`) sit behind `OssifiableProxy`:

```solidity
constructor(address impl, address admin, bytes memory initData)
    ERC1967Proxy(impl, initData) { _changeAdmin(admin); }   // delegatecalls initData on deploy
```

The proxy delegatecalls `initData` (the encoded `initialize(...)`) inside its own constructor, so initialization runs in the **same transaction** as deployment — there is no uninitialized window, and the `initializer` guard is consumed at construction. Each implementation's constructor also calls `_disableInitializers()`, so the logic contract can never be initialized directly. Per-vault `StakingVault` and `Dashboard` are deployed **and** initialized by `VaultFactory.createVaultWithDashboard*` in a single transaction. `VaultFactory` itself is non-upgradeable (no initializer). Implementation upgrades (where applicable) go through the proxy admin's `proxy__upgradeTo*`, a separate governance action.
