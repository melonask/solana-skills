# solana-skills

A specialized skill for interacting with Solana blockchains using `solana-client` 3.x/4.x and `solana-sdk` 4.x.

## Overview

Unlike EVM networks which offer `eth_getLogs` for sweeping events, Solana requires polling account signatures and parsing transaction metadata to extract Native SOL and SPL Token transfers. This skill provides the exact patterns required for fetching, parsing, and mapping Solana transactions into relational database ledgers.

## Installation

```bash
npx skills add melonask/solana-skills
```

## What This Skill Covers

- **RPC Connections:** Using `solana_client::nonblocking::rpc_client::RpcClient`.
- **Signature Sweeping:** Polling `get_signatures_for_address_with_config`.
- **Transaction Parsing:** Decoding `EncodedConfirmedTransactionWithStatusMeta`.
- **Ledger Mapping:** Using `account_index` or instruction indices as unique `event_idx` to prevent double-crediting in SQL databases.

## File Structure

```
solana/
└── SKILL.md                          # Main skill file (RPC connections, signature sweeping, parsing)
```

## License
Provided as-is for development with LLM assistants.
