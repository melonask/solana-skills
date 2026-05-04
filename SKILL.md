---
name: solana
description: "Guide for Solana blockchain interactions using solana-client v3.x and solana-sdk v4.x. Use this skill whenever the user wants to: poll Solana for incoming transactions; fetch signatures for an address; parse SPL token transfers or native SOL transfers; extract pre and post token balances; connect to a Solana RPC; or map Solana transactions to database ledgers. Make sure to use this skill when the user mentions Solana RPC, solana-client, get_signatures_for_address, SPL tokens, or tracking Solana deposits."
---

# Solana RPC & Transaction Sweeping (v3.x / v4.x)

To detect incoming deposits on Solana (which lacks EVM-style `eth_getLogs` for arbitrary contracts), the standard pattern is to poll `get_signatures_for_address` for specific deposit addresses, then fetch the parsed transactions to calculate balance deltas.

## Dependency Setup

```toml
[dependencies]
solana-client = "3.1"
solana-sdk = "4.0"
solana-commitment-config = "3"
solana-transaction-status = "3"
spl-token = "9.0"
tokio = { version = "1", features =["full"] }
```

## Core Patterns at a Glance

### 1. Connecting to RPC

```rust
use solana_client::nonblocking::rpc_client::RpcClient;
use std::sync::Arc;

let rpc_url = "https://api.mainnet-beta.solana.com";
let client = Arc::new(RpcClient::new(rpc_url.to_string()));
```

### 2. Sweeping Signatures for an Address

Fetch recent signatures for a specific address. Store the latest processed signature in your database to act as the `until` cursor for the next poll.

```rust
use solana_sdk::pubkey::Pubkey;
use solana_client::rpc_client::GetConfirmedSignaturesForAddress2Config;
use solana_sdk::signature::Signature;
use solana_commitment_config::CommitmentConfig;
use std::str::FromStr;

async fn sweep_deposits(client: &RpcClient, space_address: &Pubkey, last_processed_sig: Option<Signature>) {
    let config = GetConfirmedSignaturesForAddress2Config {
        before: None,
        until: last_processed_sig,
        limit: Some(100),
        commitment: Some(CommitmentConfig::finalized()),
    };

    let signatures = client
        .get_signatures_for_address_with_config(space_address, config)
        .await
        .unwrap();

    for sig_info in signatures {
        // Skip failed transactions
        if sig_info.err.is_none() {
            let sig = Signature::from_str(&sig_info.signature).unwrap();
            process_transaction(client, &sig, space_address).await;
        }
    }
}
```

### 3. Parsing Token/SOL Transfers (The Event Index)

When saving transactions to a SQL ledger, you need a `UNIQUE (tx, event_idx)` constraint to prevent double-crediting. In Solana, use the `account_index` or instruction index as the `event_idx`.

```rust
use solana_client::rpc_config::RpcTransactionConfig;
use solana_commitment_config::CommitmentConfig;
use solana_transaction_status::{UiTransactionEncoding, EncodedConfirmedTransactionWithStatusMeta};

async fn process_transaction(client: &RpcClient, sig: &Signature, my_address: &Pubkey) {
    let config = RpcTransactionConfig {
        encoding: Some(UiTransactionEncoding::JsonParsed),
        commitment: Some(CommitmentConfig::finalized()),
        max_supported_transaction_version: Some(0),
    };

    let tx: EncodedConfirmedTransactionWithStatusMeta = client
        .get_transaction_with_config(sig, config)
        .await
        .unwrap();

    let meta = tx.transaction.meta.unwrap();

    // SPL Token Deltas
    if let (Some(pre_tokens), Some(post_tokens)) = (meta.pre_token_balances, meta.post_token_balances) {
        for post in post_tokens {
            // Check if the balance belongs to our target address
            if post.owner == Some(my_address.to_string()) {
                let post_amt: u64 = post.ui_token_amount.amount.parse().unwrap();
                
                // Find matching pre_balance
                let pre_amt: u64 = pre_tokens.iter()
                    .find(|pre| pre.account_index == post.account_index)
                    .map(|pre| pre.ui_token_amount.amount.parse().unwrap())
                    .unwrap_or(0);

                if post_amt > pre_amt {
                    let amount_received = post_amt - pre_amt;
                    let mint = post.mint;
                    
                    // In a unified SQL ledger, the `event_idx` can be the account_index 
                    // to guarantee uniqueness inside this transaction.
                    let event_idx = post.account_index as i32;
                    
                    println!("Received {} of token {} in tx {} (idx: {})", 
                        amount_received, mint, sig, event_idx);
                }
            }
        }
    }
}
```
