# Known Issues — Solana Skill

Issues discovered through practical compilation testing against actual crate versions (solana-client 3.1.x, solana-sdk 4.0.x).

## 1. `solana_sdk::commitment_config` Not Re-Exported in v4.x (BREAKING)

The `CommitmentConfig` type is no longer in `solana_sdk::commitment_config`. It is now in a separate crate:

```rust
// WRONG (skill's code):
use solana_sdk::commitment_config::CommitmentConfig;

// CORRECT:
// Add to Cargo.toml: solana-commitment-config = "3"
use solana_commitment_config::CommitmentConfig;
```

## 2. `solana_transaction_status` as Separate Crate

The `UiTransactionEncoding` and `EncodedConfirmedTransactionWithStatusMeta` types require the `solana-transaction-status` crate as a separate dependency in v4.x:

```toml
[dependencies]
solana-transaction-status = "3"
```

## 3. Version Notes

- `solana-client = "3.1"` — **Correct.** Solana v4.x is pre-release (`4.0.0-beta.7`) and has incompatible sub-crate versions (`solana-transaction-status-client-types` conflicts). Stay on 3.1 for stable.
- `spl-token = "4.0"` → **Updated to `"9.0"`.** The latest stable SPL token crate is v9.x, fully backward-compatible with solana-client 3.1.
- `solana-sdk = "4.0"` — **Correct.** Latest stable is 4.0.1.
