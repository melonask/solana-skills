---
name: solana
description: "Comprehensive guide for Solana blockchain interactions, x402 V2 Protocol verification, and complete E2E testing pipelines using Agave v3.x/v4.x. Use this skill whenever the user wants to: automate native localnet installation; provision mock SPL tokens; write full-stack E2E tests in Rust; verify an x402 protocol payment signature; or extract pre/post token balances. IMPORTANT: For E2E testing on Apple Silicon, NEVER use Docker due to Rosetta AVX panics; ALWAYS install and run Agave natively."
---

# Solana RPC, x402 V2 Verification & Full E2E Testing (Agave v3.x/v4.x)

This skill defines the complete pipeline for testing Solana applications locally without Docker. It covers native validator installation, CLI token provisioning, and a fully functional Rust E2E test simulating an x402 HTTP handshake.

## 1. Native Installation & Localnet Automation (Bash)

Because Apple Silicon Docker containers panic on AVX instructions during Genesis Block creation, your E2E pipelines (and CI/CD scripts) must install the Agave binaries natively and start the validator in the background.

**Important**: Always export the Agave binary path before running CLI commands.

Provide the LLM or developer with this automation script (`setup_e2e.sh`):

```bash
#!/bin/bash
set -e

# Export Agave binary path (required for spl-token and solana CLI commands)
export PATH="/Users/$(whoami)/.local/share/solana/install/active_release/bin:$PATH"

echo "1. Installing Agave (Solana v3/v4) Natively..."
sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"

echo "2. Starting Localnet in Background..."
killall solana-test-validator || true
solana-test-validator --reset > validator.log 2>&1 &
sleep 10 # Wait for Genesis Block creation (increase if your system is slow)

echo "3. Configuring CLI to Localhost..."
solana config set --url http://127.0.0.1:8899
solana config set --commitment confirmed

echo "4. Provisioning E2E Test Environment..."
# Generate test keypairs
solana-keygen new --outfile merchant.json --no-bip39-passphrase --force
solana-keygen new --outfile agent.json --no-bip39-passphrase --force

# Airdrop SOL to all keypairs (including default keypair for transaction fees)
echo "Airdropping SOL..."
solana airdrop 10 $(solana-keygen pubkey agent.json)
solana airdrop 10 $(solana-keygen pubkey merchant.json)
solana airdrop 5 # Airdrop to default keypair for fee payment

# Create Mock USDC token (6 decimals, matching real USDC)
echo "Creating Mock USDC token..."
MOCK_USDC=$(spl-token create-token --decimals 6 | grep "Creating token" | awk '{print $3}')
echo "Mock USDC Mint: $MOCK_USDC"

# Create token accounts for merchant and agent (use default keypair as fee payer)
echo "Creating token accounts..."
spl-token create-account $MOCK_USDC --owner merchant.json
spl-token create-account $MOCK_USDC --owner agent.json

# Mint 100 Mock USDC to agent's token account
echo "Minting tokens to agent..."
AGENT_TOKEN_ACCOUNT=$(spl-token address --token $MOCK_USDC --owner agent.json --verbose | grep "Associated Token Address" | awk '{print $NF}')
spl-token mint $MOCK_USDC 100 $AGENT_TOKEN_ACCOUNT

echo "✅ Environment Ready!"
# Save environment variables for Rust test
echo "MOCK_USDC_MINT=$MOCK_USDC" > .env
echo "MERCHANT_PUBKEY=$(solana-keygen pubkey merchant.json)" >> .env
echo "AGENT_KEYPAIR_PATH=agent.json" >> .env
cat .env
```

## 2. Rust Dependencies

Use these latest verified compatible versions (May 2026):

```toml
[dependencies]
solana-client = "4.0"
solana-sdk = "4.0"
solana-commitment-config = "3.1"
solana-transaction-status = "3.1"
spl-token = "9.0"
spl-associated-token-account = "3.0"
tokio = { version = "1", features = ["full"] }
dotenv = "0.15"
```

**Note**: If you encounter build errors related to `spl-pod` or `solana-program`, run `cargo update` to sync to latest compatible versions.

## 3. The Full Rust E2E Test Suite

This complete Rust test verifies the full x402 lifecycle:
1. The **Agent** creates a Solana transaction and transfers Mock USDC to the Merchant.
2. The **Server** intercepts the `PAYMENT-SIGNATURE`, queries the local RPC, and verifies the pre/post token balance deltas.

Create `src/main.rs` (or `tests/e2e_tests.rs` for integration tests):

```rust
#[cfg(test)]
mod e2e_tests {
    use solana_client::nonblocking::rpc_client::RpcClient;
    use solana_client::rpc_config::RpcTransactionConfig;
    use solana_commitment_config::CommitmentConfig;
    use solana_transaction_status::{UiTransactionEncoding, EncodedConfirmedTransactionWithStatusMeta};
    use solana_sdk::{
        signature::{Keypair, Signature, Signer},
        pubkey::Pubkey,
        transaction::Transaction,
    };
    use std::str::FromStr;
    use std::sync::Arc;
    use std::env;

    /// SERVER LOGIC: Verifies the exact payment amount hit the Merchant's account
    async fn verify_x402_payment(
        client: &RpcClient, 
        signature_str: &str, 
        merchant_pubkey: &str, 
        expected_amount_decimals: u64
    ) -> Result<bool, Box<dyn std::error::Error>> {
        let sig = Signature::from_str(signature_str)?;
        let config = RpcTransactionConfig {
            encoding: Some(UiTransactionEncoding::JsonParsed),
            commitment: Some(CommitmentConfig::confirmed()),
            max_supported_transaction_version: Some(1),
        };

        let tx: EncodedConfirmedTransactionWithStatusMeta = client
            .get_transaction_with_config(&sig, config)
            .await?;

        let meta = tx.transaction.meta.unwrap();
        if meta.err.is_some() { 
            return Ok(false); 
        }

        if let (Some(pre_tokens), Some(post_tokens)) = (meta.pre_token_balances, meta.post_token_balances) {
            for post in post_tokens {
                if post.owner == Some(merchant_pubkey.to_string()) {
                    let post_amt: u64 = post.ui_token_amount.amount.parse().unwrap_or(0);
                    let pre_amt: u64 = pre_tokens.iter()
                        .find(|pre| pre.account_index == post.account_index)
                        .map(|pre| pre.ui_token_amount.amount.parse().unwrap_or(0))
                        .unwrap_or(0);

                    if post_amt > pre_amt {
                        let actual_received = post_amt - pre_amt;
                        if actual_received == expected_amount_decimals {
                            return Ok(true);
                        }
                    }
                }
            }
        }
        Ok(false)
    }

    #[tokio::test]
    async fn test_full_x402_flow() {
        dotenv::dotenv().ok();
        
        let rpc_url = "http://127.0.0.1:8899";
        let client = Arc::new(RpcClient::new(rpc_url.to_string()));

        let mint_pubkey = Pubkey::from_str(&env::var("MOCK_USDC_MINT").unwrap()).unwrap();
        let merchant_pubkey = Pubkey::from_str(&env::var("MERCHANT_PUBKEY").unwrap()).unwrap();
        
        let agent_keypair_path = env::var("AGENT_KEYPAIR_PATH").unwrap();
        let agent_wallet = solana_sdk::signature::read_keypair_file(&agent_keypair_path).unwrap();

        let agent_ata = spl_associated_token_account::get_associated_token_address(&agent_wallet.pubkey(), &mint_pubkey);
        let merchant_ata = spl_associated_token_account::get_associated_token_address(&merchant_pubkey, &mint_pubkey);

        let invoice_amount: u64 = 1_000_000; // 1.0 USDC (6 decimals)

        let transfer_ix = spl_token::instruction::transfer(
            &spl_token::id(),
            &agent_ata,
            &merchant_ata,
            &agent_wallet.pubkey(),
            &[&agent_wallet.pubkey()],
            invoice_amount,
        ).unwrap();

        let recent_blockhash = client.get_latest_blockhash().await.unwrap();
        let tx = Transaction::new_signed_with_payer(
            &[transfer_ix],
            Some(&agent_wallet.pubkey()),
            &[&agent_wallet],
            recent_blockhash,
        );

        let signature = client.send_and_confirm_transaction(&tx).await.unwrap();
        println!("🤖 AGENT: Payment Sent! Signature: {}", signature);

        let is_valid = verify_x402_payment(
            &client, 
            &signature.to_string(), 
            &merchant_pubkey.to_string(), 
            invoice_amount
        ).await.unwrap();

        assert!(is_valid, "❌ x402 Server Verification Failed!");
        println!("✅ SERVER: Payment Verified. E2E Test Passed!");
    }
}
```

## 4. Running the Test

1. Run the setup script to provision the local environment:
```bash
chmod +x setup_e2e.sh && ./setup_e2e.sh
```

2. Initialize a Rust project:
```bash
cargo init --name solana_e2e_test
```

3. Copy dependencies to `Cargo.toml` and test code to `src/main.rs`

4. Run the test:
```bash
export PATH="/Users/$(whoami)/.local/share/solana/install/active_release/bin:$PATH"
cargo test test_full_x402_flow -- --nocapture
```

## 5. Troubleshooting

- **Validator not starting**: Check `validator.log` for errors
- **PATH errors**: Always export Agave binary path before CLI commands
- **Airdrop failures**: Wait a few seconds and retry due to rate limits
- **Build errors**: Run `cargo update` to sync dependencies
