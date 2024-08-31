---
title: "Solana Token Transfer Indexer: Design"
date: 2024-08-31T00:00:00+00:00
description: "A design doc for token history indexer"
tags: ["solana", "rust", "transaction-indexer", "radar-hackathon"]
type: post
weight: 25
showTableOfContents: true
---

## Introduction

In order to design an token transaction indexer, I will compare two existing indexers currently in operation at Helius: DAS (Digital Asset RPC Infrastructure) and Photon and cherry pick some of its features.

- **DAS** is used to index all digital assets on Solana, including Tokens, NFTs, and cNFTs. You can explore more about DAS [here](https://github.com/helius-labs/digital-asset-rpc-infrastructure).
- **Photon** is a new indexer launched by Helius to store compressed accounts. More details can be found [here](https://docs.helius.dev/zk-compression-and-photon-api/what-is-zk-compression-and-photon).

## DAS vs Photon 

### Streaming Mechanism
- **DAS**: Uses a custom Geyser plugin to stream accounts and transactions into its indexer. This plugin is specific to DAS and can be found [here](https://github.com/helius-labs/digital-asset-validator-plugin).
- **Photon**: Utilizes the [yellowstone-grpc](https://github.com/helius-labs/yellowstone-grpc) gRPC Geyser plugin to stream accounts and transactions.

### Message Buffering
- **DAS**: Uses Redis streams as a message buffer, necessitating the running of a Redis server.
- **Photon**: Leverages Rust's stream primitives to stream data from Geyser. It fetches blocks concurrently and indexes them serially.

### Data Backfilling
- **DAS**: Requires a Geyser restart to perform data backfilling.
- **Photon**: Employs polling via RPC calls and auto-healing techniques to fill gaps automatically.

### Code Structure
- **DAS**: Has multiple binaries, one for the indexer and another for the API.
- **Photon**: Consolidates functionality into a single binary, which can be run in multiple modes.

### Database and APIs 
- **Database**: Both use Postgres as their database.
- **API**: Both use JSON RPCs for their APIs.

## Design Choices for the Transaction Indexer

### Streaming Mechanism
- The Transaction Indexer will use the gRPC Geyser plugin to stream transactions from the TokenKeg and Token2022 programs. This is a natural choice given the existing infrastructure.

### Polling and Backfilling
- Like Photon, the Transaction Indexer will use polling via RPC as a fallback mechanism to fill in any gaps in the data.

### Message Passing
- Given the expected volume of updates, I will use `mpsc` channels for message passing between the polling logic and the indexing logic. This approach is similar to using Redis streams and will help minimize indexing latencies.

### Code Structure
- The indexer will be split into two binaries: one for indexing and another for the API. This separation keeps the API and indexing logic isolated and easier to manage.

### Database Choice
- I will use **TimescaleDB** instead of vanilla Postgres. The reasons for this choice are detailed in the next section.

### API Format
- Like both DAS and Photon, the Transaction Indexer will use JSON RPCs.

## TimescaleDB

### What is TimescaleDB?
TimescaleDB is a time-series database built on top of Postgres. It offers powerful features like automatic partitioning, compression, and scaling, making it well-suited for handling time-series data.

### Why Use TimescaleDB for the Transaction Indexer?
- **Performance**: TimescaleDB's optimizations for time-series data make it ideal for indexing transactions, which inherently involve time-based data.
- **Scalability**: As transaction volume grows, TimescaleDB can scale efficiently, handling large datasets with ease.
- **Flexibility**: TimescaleDB allows for preserving history for a defined period (e.g., 3 months) while still enabling custom access to data from the genesis block.

### Why Not Use ClickHouse?
- While [ClickHouse](https://clickhouse.com/) is another popular choice for time-series data, TimescaleDB's integration with Postgres provides a smoother transition for projects already using Postgres. Additionally, TimescaleDB's native support for SQL and compatibility with Postgres tooling is advantageous.

I also referred this doc[https://www.timescale.com/blog/what-is-clickhouse-how-does-it-compare-to-postgresql-and-timescaledb-and-how-does-it-perform-for-time-series-data/] which compares the performance of Timescale DB and Clickhouse and based on their findings, I will be using Timescale DB


### DB schema 

### Blocks Table
- **Purpose**: Tracks the blocks indexed and identifies gaps for automatic backfilling.

- **Columns**:
  - `slot`: bigint
  - `parent_slot`: bigint
  - `block_height`: bigint
  - `block_time`: bigint

- **Indexes**:
  - Primary Key: `slot, block_time`

### Token Transfers Table
- **Purpose**: Indexes all token transfer instructions and associated data.
- **Columns**:
  - `signature`: bytea
  - `source_address`: bytea
  - `program_id`: bytea
  - `destination_address`: bytea
  - `source_ata`: bytea (nullable)
  - `destination_ata`: bytea (nullable)
  - `mint_address`: bytea (nullable)
  - `slot`: bigint
  - `amount`: bigint
  - `error`: text (nullable)
  - `block_time`: timestamp with time zone
  - `created_at`: timestamp without time zone

- **Indexes**:
  - Primary Key: `signature, source_address, destination_address, block_time`

The token_transfers table records token transfer transactions. It includes the signature of the transaction, the addresses involved (source_address, destination_address), and other details like program_id, slot, amount, and any potential error messages. The table also records the slot and block_time timestamp of when the transfer was created.

The `blocks` table is used to keep track of indexed blocks and to automatically backfill any gaps. The `token_transfers` table records token transfer transactions, including details like the involved addresses, program ID, slot, amount, and timestamps. The focus is on indexing only the Token and TokenExtensions programs, keeping the schema simple and targeted.

Also, the `token_transfers` table will be a hypertable and compression feature will be used to compress data older than 3 months.

### APIs 

We're going to just 1 single API to retrive transaction data

#### GetTransactionsByAddress

This endpoint retrieves a list of transactions filtered by a source address and/or destination address or mint address. The results can be paginated and sorted based on the provided parameters.

Request
```
{
    "jsonrpc": "2.0",
    "id": "0",
    "method": "get_transactions_by_address",
    "params": {
        "source": "string",                     // Public key of the source address (optional)
        "destination": "string",                // Public key of the destination address (optional)
        "mint": "string",                       // Public key of the mint address (optional)
        "limit": "u32",                         // Limit on the number of transactions returned (optional)
        "page": "u32",                          // Page number for pagination (optional)
        "before": "string",                     // Filter transactions before this date (dd/mm/yyyy format, optional)
        "after": "string",                      // Filter transactions after this date (dd/mm/yyyy format, optional)
        "sort_by": {                            // Sorting options (optional)
            "sort_by": "TransactionSortBy",     // Field to sort by, default is "slot"
            "sort_direction": "TransactionSortDirection" // Sorting direction, default is "desc"
        }
    }
}
```

Response
```
{
    "jsonrpc": "2.0",
    "result": {
        "total": "u32",                        // Total number of transactions found
        "limit": "u32",                        // Limit on the number of transactions returned
        "page": "u32",                         // Current page number
        "items": [                             // List of transaction items
            {
                "signature": "string",          // Transaction signature
                "source_address": "string",     // Public key of the source address
                "program_address": "string",    // Public key of the program address
                "destination_address": "string",// Public key of the destination address
                "source_ata": "string",         // Associated token account of the source address
                "destination_ata": "string",    // Associated token account of the destination address
                "mint_address": "string",       // Public key of the mint address
                "slot": "u64",                  // Slot number of the transaction
                "amount": "u64",                // Amount transferred in the transaction
                "block_time": "string"          // Block time of the transaction (ISO 8601 format)
            }
        ]
    },
    "id": "0"
}
```