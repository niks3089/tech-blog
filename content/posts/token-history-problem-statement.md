---
title: "Solana Token History Indexer: Problem Statement"
date: 2024-08-10T17:55:28+08:00
description: "A requirement for token history indexer"
tags: ["solana"]
type: post
weight: 25
showTableOfContents: true
---

# Problem Statement: Token Transfer History Indexer for Wallets

### Background
Retrieving transaction history for a specific wallet on the Solana blockchain, especially for token transfers, currently requires multiple RPC calls. This process is slow and inefficient, making it challenging to quickly access and filter token transfer data. There is a clear need for a streamlined and optimized method to retrieve token transfer history.

Here's an example of how to fetch token transfer history specifically between two wallets

```rust
use solana_client::rpc_client::RpcClient;
use solana_sdk::signature::Signature;
use solana_transaction_status::{EncodedTransaction, UiTransactionEncoding};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Set up the RPC client
    let rpc_url = "https://api.mainnet-beta.solana.com";
    let client = RpcClient::new(rpc_url);

    // Specify the source and destination wallet addresses
    let source_wallet = "SourceWalletAddress";
    let destination_wallet = "DestinationWalletAddress";
    
    // Fetch the transaction signatures for the source wallet address
    let signatures = client.get_signatures_for_address_with_config(
        &source_wallet.parse()?,
        solana_client::rpc_request::RpcSignaturesForAddressConfig {
            limit: Some(1000),
            before: None,
            until: None,
            commitment: None,
        },
    )?;

    // Iterate over the transaction signatures and fetch the details
    for signature in signatures {
        let transaction_signature = Signature::from_str(&signature.signature)?;
        let transaction = client.get_transaction_with_config(
            &transaction_signature,
            solana_transaction_status::UiTransactionEncoding::JsonParsed,
        )?;

        // Parse the transaction instructions to identify token transfers
        if let Some(meta) = &transaction.transaction.meta {
            for instruction in &meta.inner_instructions {
                for ui_instruction in &instruction.instructions {
                    // Filter for token transfer instructions by program ID
                    if ui_instruction.program_id == "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA" {
                        if let solana_transaction_status::UiInstruction::Parsed(parsed_instruction) = ui_instruction {
                            // Check if this is a transfer instruction
                            if let Some(parsed) = &parsed_instruction.parsed {
                                if let solana_transaction_status::UiInstructionParsed::Transfer { source, destination, amount } = parsed {
                                    // Only print if the transfer is between the specified wallets
                                    if source == source_wallet && destination == destination_wallet {
                                        println!(
                                            "Transfer: {} SOL from {} to {}",
                                            amount, source, destination
                                        );
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    Ok(())
}

```

### Objective
Develop an indexer that efficiently captures and stores token transfer histories for specific wallets across various token types (e.g., token extensions, SPL tokens, Metaplex tokens). This indexer will enable quick and efficient querying of token transfer histories, reducing the dependency on multiple RPC calls.

### Requirements

#### Data Retention
- Support at least 30 days of token transfer history, starting from a specified launch date (e.g., October 1st).
- Plan for potential extensions of retention periods to 60-90 days.

#### Token Compatibility
- Ensure compatibility with all uncompressed token types, including token extensions, regular tokens, and Metaplex tokens.

#### Performance
- Optimize the indexer for speed and filtering capabilities.
- Minimize the need for multiple RPC calls to retrieve token transfer histories.

#### Storage Solution
- Utilize TimescaleDB for efficient time-series data management.
- Design the database schema to handle high volumes of token transfer data efficiently.

#### API Development
- Develop an API to provide access to the indexed token transfer history.
- Support querying token transfers to/from a specific wallet.
- Implement filtering options such as date ranges and specific token types.

### Scope

#### Indexer Development
- Create a robust indexer to monitor and record token transfers from the blockchain.
- Ensure the indexer starts recording data from a specified date (e.g., October 1st).

#### Database Schema Design
- Design a schema in TimescaleDB to store token transfer data efficiently.
- Implement partitions or hypertables for optimized querying and retention management.

#### API Implementation
- Develop JSON RPC endpoints to query token transfer history.
- Endpoints should support filtering by wallet address, token type, and date range.

#### Monitoring and Maintenance
- Implement monitoring to ensure the indexer is capturing data correctly.
- Develop maintenance scripts to manage data retention policies.

### Deliverables

#### Indexer
- A fully functional indexer that captures token transfer data from the blockchain.

#### Database Schema
- A TimescaleDB schema designed for efficient storage and querying of token transfer data.

#### API
- A set of JSON RPC API endpoints to access token transfer history.
- Documentation for API usage and querying capabilities.

#### Monitoring and Maintenance Tools
- Scripts and tools for monitoring the indexer's performance and maintaining data retention policies.

### Expected Challenges

#### Data Volume
- Handling high volumes of token transfer data while ensuring the indexer remains performant.

#### Compatibility
- Ensuring compatibility with various token standards and types.

#### Retention Management
- Efficiently managing data retention to comply with the specified retention period.

### Conclusion
By developing an efficient indexer and API for token transfer history, we aim to significantly improve the speed and ease of retrieving and filtering token transfer data. This solution will enhance the user experience and provide a scalable method for managing and querying blockchain transaction data.
