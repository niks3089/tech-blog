---
title: "Solana Token Transfer Indexer: Problem Statement"
date: 2024-08-18T00:00:00+00:00
description: "A requirement for token history indexer"
tags: ["solana", "rust", "transaction-indexer", "radar-hackathon"]
type: post
weight: 25
showTableOfContents: true
---

# Problem Statement: Token Transfer History Indexer 

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
    let rpc_url = "https://mainnet.helius-rpc.com/?api-key=api-key";
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
Develop an indexer and APIs that efficiently captures, stores and exposes token transfer histories across various token types (e.g., SPL tokens & Token Extensions). The APIs will enable quick and efficient querying of token transfer histories thus reducing the need to make multiple RPC calls.

### Usecases 

Here's some of the usecases where an token transfer indexer will be useful 

- Enterprise users of Token Extensions or SPL will need history of all the transactions that ever happened to fulfill their audit requirements
- Chargeback via permanent delegates will require transaction history to identify and validate frauds

### Requirements

#### Indexer 
- Parse and index token transfer transactions 
- I will start with indexing the following fields 
    - transaction signature
    - source account
    - destination account 
    - source associated token account 
    - destination associated token account
    - mint account
    - amount 
    - slot
- Support at least 90 days of token transfer history, starting from a specified launch date (e.g., October 1st).
- Customers requiring beyond the 90 days of history can reach out and can pay for the additional storage/compute costs and system should be able to handle indexing from genesis. 
- Ensure compatibility with all all token types, including token extensions & regular SPL tokens. 
- Ensure the system can be extended to easily index new token types, for example, ZK compressed tokens.
- Design the database schema to handle high volumes of token transfer data efficiently.

#### API 
- Develop multiple APIs to provide access to the indexed token transfer history. Initially, we will just have a single `GetTransactions` JSON RPC API. The request could look something like this
    ```api
     {
      "jsonrpc": "2.0",
      "id": "helius",
      "method": "GetTransactions"
      "params": {
        "sourceAccount": <source wallet>
        "destinationAccount": <destination wallet (optional)>
        "mintAccount": <mint account to filter on (optional)> 
        "from": <date in dd/mm/yy to filter on (optional)>
        "to": <date in dd/mm/yy to filter on (optional)>
      }
    }
    ```
- Ensure that the API responses are fast, reliable and correct

### Deliverables

- A fully functional indexer that captures token transfer data from the blockchain.
- A persistent DB designed for efficient storage and querying of token transfer data.
- A set of JSON RPC API endpoints to access token transfer history.

### Expected Challenges

- Indexing high volumes of token transfer data while ensuring the there no gaps when fetching transactions.
- Ensuring querying such large amount of data is fast, reliable and accurate. 
- Efficiently scaling the system as the data grows without it getting out of budget and making good tradeoffs.

### Conclusion
By developing an efficient indexer and API for token transfer history, we aim to significantly improve the speed and ease of retrieving and filtering token transfer data. This solution will enhance the user experience and provide a scalable method for managing and querying Solana transaction data.

