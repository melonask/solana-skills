# solana-skills

A specialized skill for interacting with Solana/Agave blockchains using `solana-client` v3.x and `solana-sdk` v4.x.

## Overview

Unlike EVM networks which offer `eth_getLogs` for sweeping events, Solana requires polling account signatures and parsing transaction metadata to extract Native SOL and SPL Token transfers. Furthermore, modern Web3 protocols (like x402 V2) demand precise cryptographic verification and robust End-to-End (E2E) testing environments.

This skill provides the exact patterns required for fetching and parsing transactions, verifying x402 protocol payment signatures, and automating native Agave localnet E2E testing pipelines (specifically engineered to bypass Apple Silicon Docker panics).

## Installation

```bash
npx skills add melonask/solana-skills
```

## What This Skill Covers

- **Native E2E Localnet Automation:** Bash scripts to natively install Agave, spin up `solana-test-validator`, and provision mock SPL tokens (avoiding Rosetta 2 AVX Docker panics on M-series Macs).
- **x402 V2 Protocol Verification:** Mathematically verifying exact pre/post SPL token balances for a single `PAYMENT-SIGNATURE` payload.
- **Full Rust E2E Testing:** Using `#[tokio::test]` to simulate both the Client (executing transfers) and the Server (intercepting and verifying transactions via RPC).
- **RPC Connections:** Utilizing modern asynchronous `solana_client::nonblocking::rpc_client::RpcClient`.
- **Signature Sweeping:** Polling `get_signatures_for_address_with_config` for background ledger synchronization.
- **Ledger Mapping:** Using `account_index` as a unique `event_idx` to prevent double-crediting in SQL databases.

## File Structure

```text
solana/
└── SKILL.md                          # Main skill file (Localnet automation, x402 verification, E2E Rust tests, and ledger sweeping)
```

## License
Provided as-is for development with LLM assistants.
