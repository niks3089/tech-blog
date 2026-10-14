---
title: "Building a Self-Hosted Solana Portfolio Tracker with an Encrypted Vault"
date: 2026-10-14T11:00:00+05:30
description: "How I built a multi-wallet Solana tracker that derives cost basis from swap history, computes XIRR, and stores your data encrypted under a key derived from a wallet signature."
tags: ["solana", "defi", "typescript", "projects"]
type: post
weight: 25
showTableOfContents: true
---

## The problem

I hold assets across several Solana wallets, some of them in DeFi protocols. Most dashboards show what each wallet holds right now. They don't answer what I care about: what did I pay for this, and how is the whole thing doing as an investment?

There was also a trust problem. Grouping wallets into a "household" tells a server which addresses belong together. I didn't want any server, mine included, holding that mapping in plaintext.

So I built [portfolio](https://portfolio.niks3089.com), a self-hosted, MIT-licensed tracker. The source is on [GitHub](https://github.com/niks3089/solana-portfolio-tracker).

## What it does

- Token holdings and DeFi positions (Kamino, Drift, Meteora, Exponent and others).
- Trade-based P&L, with cost basis derived from on-chain swap history.
- A returns row: Current Value, Invested, Absolute Return and XIRR.
- A trade history table with every priced buy and sell.
- Portfolios: named groups of wallets tracked together, with a daily net-worth chart.
- Display currencies, CSV export, and an optional Backpack Exchange connection.

You don't need to connect a wallet to use it. Paste any address or `.sol` name and browse. Connecting is only needed to save portfolios, because those live in the encrypted vault.

## Architecture

The stack is small on purpose: a React 19 + Vite + Tailwind v4 SPA, a Node 22 + Express + TypeScript server, and one SQLite file via better-sqlite3.

```text
 Browser (React SPA)                      Server (Express)                 Upstream APIs
+------------------------------+         +---------------------------+
| wallet.signMessage()         |         | /api/portfolio/*          |---> Birdeye  (balances, prices,
|   -> SHA-256 -> AES-256-GCM  |  HTTPS  |   holdings, DeFi,         |              historical prices)
| encrypt / decrypt vault blob |<------->|   trade-pnl, fx           |---> Helius   (enhanced txs)
| Backpack Ed25519 signing     |         |                           |---> Lambda / Dialect (DeFi)
+------------------------------+         | /api/portfolio/backpack --|---> Backpack (signed reads)
                                         |                           |
                                         | /api/vault/*  (no crypto) |
                                         |   opaque blob store ------|---> SQLite: owner_wallet,
                                         +---------------------------+     ciphertext, iv, version
```

The server has two jobs. It proxies data providers so API keys never reach the browser, and it stores one encrypted blob per wallet that it cannot read.

One detail from production: any Birdeye price on a holding worth $25 or more gets cross-checked against Jupiter's price API. Tokens in dead pools were showing phantom values and inflating net worth.

## How P&L and cost basis work

This took the most iteration. It all lives in `server/services/trade-pnl.ts`.

### Reading swaps

For each wallet the server pulls up to 2,000 transactions from Helius's Enhanced Transactions API and caches them for an hour. Each swap's legs are split into cash and tokens. USDC, USDT and PYUSD are cash at $1. SOL is cash at the current SOL price. Everything else is a token.

Cash out with tokens in is a buy. Tokens out with cash in is a sell. Token-for-token swaps with no cash leg don't create cost basis.

Some transactions are typed `SWAP` but have no parsed swap event. For those I rebuild the swap from the raw transfers, ignoring SOL movements under 0.001 SOL so fees don't look like trades.

### Transfers in

Tokens often arrive by transfer. If a wallet received a token it still holds from an address outside the portfolio, the server prices it with Birdeye's historical price for that day and adds it to cost basis.

Transfers between wallets in the same portfolio are skipped, so cost basis follows tokens around the household. Each row uses the wallet's own cost history first. If there is none, say the token was bought in wallet A and moved to B, it falls back to the portfolio-wide average.

### Average cost

I use average cost, not FIFO:

```text
avg_cost   = total_spent / max(total_bought, total_held)
cost_basis = current_balance * avg_cost
pnl        = current_balance * current_price - cost_basis
```

The `max` matters. After a partial sell you hold less than you bought, so this is the normal average. If you hold more than the history explains (rewards, airdrops, buys older than the scan window), the known cost is spread over the whole balance, so the extra tokens count as zero cost. The result is unrealized P&L on what you hold today.

### XIRR

Absolute return ignores time, so the returns row also shows XIRR.

The key decision was what counts as a cashflow. Swaps don't, because they only move value inside the portfolio. Transfers across the household boundary do: money in is negative, money out is positive, each priced on its day. The final cashflow is today's net worth, including DeFi deposits minus borrows.

The solver runs Newton-Raphson from 10%, then falls back to bisection between -99% and 10,000% if that doesn't converge. Release 1.1.0 fixed a bug where the terminal value left out DeFi positions, which understated XIRR.

The server also computes a benchmark: the XIRR if the same cashflows had all gone into SOL. The UI doesn't show it yet.

## The encrypted vault

Portfolios, wallet names, snapshots and Backpack credentials go into one JSON object that's encrypted in the browser before it goes anywhere.

### Key derivation

The browser asks the wallet to sign a fixed message:

```text
solana-portfolio:vault:v1:<pubkey>
```

Ed25519 signatures are deterministic (RFC 8032), so the same wallet signing the same message always produces the same bytes. SHA-256 of that signature is the AES-256-GCM key. There's no seed to store and no password to forget. Each write uses a fresh 12-byte IV.

The key is cached in `sessionStorage` under `vault.aesKey:<pubkey>`. An earlier version used one shared slot, and switching wallets could pick up the wrong key. That was a real bug.

### Authenticating writes

The first version had a hole. Anyone who knew your pubkey could read your vault's version number and PUT garbage over it. They couldn't read your data, but they could destroy it.

The fix is a second signature over a different, timestamped message:

```text
solana-portfolio:vault-auth:v1:<pubkey>:<unix_ts>
```

The server checks it with Node's built-in Ed25519 verification, rejects stale or future timestamps, and issues a 24-hour JWT bound to that wallet. `PUT` requires the token.

The two messages must stay separate. If the server ever saw the key-derivation signature, it could compute the AES key. So that one never leaves the browser. Unlocking only asks for the key signature; the auth signature is requested on the first save.

### What the server sees

The `vaults` table has five columns: `owner_wallet`, `ciphertext`, `iv`, `version` and `updated_at`. The vault code is two short files, `server/vault.ts` and `server/routes/vault.ts`, and neither imports a crypto library. The README has grep commands to check that.

`version` is an optimistic concurrency counter. If another tab saved first, the server returns 409 instead of silently overwriting.

### Threat model

- Dumping the SQLite file gets you ciphertext and pubkeys. The pubkey is plaintext because it's the lookup key.
- A leaked JWT secret lets an attacker forge write tokens, but not read anything.
- An operator who serves modified JavaScript can steal the key after you sign. Every web app has this boundary. A strict CSP narrows the exfiltration paths, but the real answers are checking the shipped JS or self-hosting.
- Addresses you look up still pass through the server to Helius and Birdeye.

## Backpack without trusting the server

Backpack Exchange has no CORS, so the server has to proxy. I didn't want it holding exchange secrets.

The API secret lives only in the encrypted vault. The browser imports it as a WebCrypto Ed25519 key and signs two read instructions, `balanceQuery` and `collateralQuery`, with a 5-second window. The server verifies both signatures, forwards them, and caches nothing per user. It never sees the secret, so it can't sign anything else.

## Tradeoffs I accepted

- **SOL legs use today's price.** Swaps paid in SOL are valued at the current SOL price. Transfers use historical prices; swaps don't yet.
- **History is capped** at 2,000 transactions per wallet. Older activity drops out of cost basis.
- **No key rotation.** The key is a pure function of your wallet, so losing the wallet means losing the vault. The `v1` in the message leaves room to change that.
- **Snapshots need you.** The chart records at most one point per portfolio per day, keeps 180, and only while the dashboard is open and unlocked. There's no background job.
- **Boring infrastructure.** One Node process, one SQLite file. A backup is as opaque as the live database.

## What's next

There's no formal roadmap, but these are the gaps I want to close:

- Historical SOL prices for SOL-denominated swaps.
- Showing the SOL benchmark XIRR the server already computes.
- A test suite. There isn't one yet, and the P&L code needs it most.

If you want to poke at the cost-basis math or check the vault claims, the code is on [GitHub](https://github.com/niks3089/solana-portfolio-tracker), and the hosted instance is at [portfolio.niks3089.com](https://portfolio.niks3089.com).
