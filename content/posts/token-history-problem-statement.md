---
title: "Solana Token History Indexer: Problem Statement"
date: 2024-08-11T17:55:28+08:00
description: "A requirement for token history indexer"
tags: ["solana", "rust"]
type: post
weight: 25
showTableOfContents: true
---

# Problem Statement: Token Transfer History Indexer for Wallets

### Background
Retrieving transaction history for a specific wallet or multiple wallets on the Solana blockchain, especially for token transfers, currently requires multiple RPC calls and complicated filtering. This process is slow and inefficient, making it challenging to quickly access and filter token transfer data. There is a clear need for a streamlined and optimized method to retrieve token transfer history.


Here's an example of how we fetch token transfer history specifically between two wallets today

```rust
use solana_client::rpc_client::RpcClient;
use solana_sdk::signature::Signature;
use solana_sdk::pubkey::Pubkey;
use solana_transaction_status::{EncodedTransaction, UiTransactionEncoding};
use spl_token::state::Account;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Set up the RPC client
    let rpc_url = "https://api.mainnet-beta.solana.com";
    let client = RpcClient::new(rpc_url);

    // Specify the source and destination wallet addresses
    let source_wallet: Pubkey = "SourceWalletAddress".parse()?;
    let destination_wallet: Pubkey = "DestinationWalletAddress".parse()?;

    // Fetch all token accounts associated with the source wallet
    let source_token_accounts = client.get_token_accounts_by_owner(
        &source_wallet,
        solana_client::rpc_request::TokenAccountsFilter::ProgramId(spl_token::id()),
    )?;

    // Iterate over all token accounts
    for token_account in source_token_accounts.value.iter() {
        let token_account_pubkey: Pubkey = token_account.pubkey.parse()?;

        // Fetch the transaction signatures for each token account address
        let signatures = client.get_signatures_for_address_with_config(
            &token_account_pubkey,
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
                                        if source == source_wallet.to_string() && destination == destination_wallet.to_string() {
                                            println!(
                                                "Transfer: {} tokens from {} to {}",
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
    }
    Ok(())
}
```
The provided code retrieves the token transfer history between two wallets on the Solana blockchain by first identifying all token accounts associated with the source wallet. It then iterates through each token account, fetching transaction signatures and the corresponding transaction details. By parsing the transaction instructions, the code filters for token transfer operations specifically involving the SPL token program. It ensures that only transfers between the specified source and destination wallets are captured, including transfers that occur through associated token accounts rather than the main wallet address. 

So to fully capture a wallet's transaction history, one must first discover all associated token accounts and then query the blockchain for each of these accounts individually. Moreover, the process involves making multiple RPC calls to fetch transaction signatures and details, parsing intricate transaction data and filtering relevant token transfers. This can be resource-intensive and slow, especially with high transaction volumes or when dealing with multiple wallets, making it challenging to efficiently gather and process comprehensive token transfer history.

### Objective
Develop an indexer that efficiently captures and stores token transfer histories for specific wallets across various token types (e.g., SPL tokens, Token Extensions and Metaplex tokens). This indexer will enable quick and efficient querying of token transfer histories, reducing the dependency on multiple RPC calls.

### Requirements

#### Data Retention
- Support at least 30 days of token transfer history, starting from a specified launch date (e.g., October 1st).
- Plan for potential extensions of retention periods to 60-90 days.

#### Token Compatibility
- Ensure compatibility with all uncompressed token types, including token extensions, regular tokens and Metaplex tokens.

#### Performance
- Optimize the indexer for speed and filtering capabilities.
- Minimize the need for multiple RPC calls to retrieve token transfer histories.

#### Storage Solution
- Utilize TimescaleDB for efficient time-series data management.
- Design the database schema to handle high volumes of token transfer data efficiently.

#### API Development
- Develop multiple APIs to provide access to the indexed token transfer history.
- Support querying token transfers to/from a specific wallet.
- Implement filtering options such as date ranges and specific token types.

### Scope

#### Indexer Development
- Create an indexer which listens and record token transfers from gRPC Geyser based validator.
- Ensure the indexer is resilient to failures and ensures data correctly and availability.

#### Database Storage 
- Design a schema using TimescaleDB to store token transfer data and retrieve it quickly and efficiently.
- Use its hypertable feature for optimized querying and data management.

#### API Implementation
- Develop JSON RPC endpoints to query token transfer history.
- Endpoints should support filtering by wallet address, token type and date range.


### Deliverables

#### Indexer
- A fully functional indexer that captures token transfer data from the blockchain.

#### Database
- A TimescaleDB schema designed for efficient storage and querying of token transfer data.

#### API
- A set of JSON RPC API endpoints to access token transfer history.
- Documentation for API usage and querying capabilities.

#### Monitoring and Maintenance Tools
- Scripts and tools for monitoring the indexer's and api performance and maintaining data retention policies.

### Expected Challenges

#### Data Volume
- Handling high volumes of token transfer data while ensuring the indexer remains performant.

#### Querying Speed
- Ensuring querying such large amount of data is fast, reliable and accurate. 

#### Retention Management
- Efficiently managing data retention to comply with the specified retention period.

### Conclusion
By developing an efficient indexer and API for token transfer history, we aim to significantly improve the speed and ease of retrieving and filtering token transfer data. This solution will enhance the user experience and provide a scalable method for managing and querying blockchain transaction data.
