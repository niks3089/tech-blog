---
title: "Brigade: A Social Network on Solana and Tapedrive"
date: 2026-10-21T01:00:00+05:30
description: "How Brigade puts identity and ordering on Solana and content on Tapedrive: a native program for accounts and keys, signed posts with no wallet popups, batch commits, hourly checkpoints, and content that expires in 3 to 6 months."
tags: ["solana", "social", "architecture", "projects"]
type: post
weight: 25
showTableOfContents: true
---

> **Status:** running on Solana devnet. Tapedrive hasn't launched on mainnet yet, so neither has Brigade. Everything stored so far is test data and may be reset.

## What Brigade is

Brigade is a social network for the Solana ecosystem: a public feed with market context, built around wallets. I first wrote this up as a pitch. Since then it's been built, and the architecture has moved a long way from that pitch.

The original plan was a conventional stack: Postgres as the source of truth, with the chain used for money. The design I ended up with is different:

- **Identity and ordering live on Solana.** Accounts, handles, posting keys and the order of posts are recorded by a small native program.
- **Content lives on Tapedrive.** Every post is a signed message stored as its own object, with its media next to it.
- **Postgres is just an index.** It makes feeds and search fast, and the goal is that it can be thrown away and rebuilt from Solana and Tapedrive.

The rest of this post covers why, how the pieces fit, what happens when you post, and what isn't done yet.

## Why put social on Solana

A social network's real assets are who you are, who said what, and when. On a normal platform all three sit in one company's database. If the company changes the rules, shuts down or quietly edits history, you can't check.

Putting those on a chain changes what users can verify:

- **Your account belongs to your wallet.** The handle is registered on-chain and owned by your wallet, not by a row Brigade can reassign.
- **Every post is signed by you.** Anyone can check that a post came from a key your wallet approved, without trusting Brigade's server.
- **The order is public.** Posts are committed to Solana in batches with a gapless sequence number, so the server can't silently insert, drop or reorder history.
- **Money is native.** Tips, swaps and token drops are Solana transactions anyway, so they sit next to the identity that makes them meaningful.

What doesn't belong on a chain is the content itself. Posts and images are large and mostly short-lived, and paying chain storage for a "gm" makes no sense. So Solana holds hashes, and the bytes go to storage built for bytes.

## Design decisions

### Decision 1: content never goes in Solana state

Solana stores identity accounts and a batch log of Merkle roots. Post bodies and media never sit in account state or transaction data. Only their hashes do.

That keeps on-chain cost proportional to the number of batches, not the number of posts, and it means moderation can remove content without fighting the chain (more on that below).

### Decision 2: one native program for identity and ordering

The program, `brigade-chain`, is a native Solana program with fixed byte layouts. It has a handful of account types:

| Account | Holds |
|---|---|
| Account | the owner wallet and the handle |
| Handle | a reverse lookup, so each handle maps to exactly one account |
| App key | a posting key the owner approved, with its expiry |
| Log | the batch sequence number and the last Merkle root |
| Checkpoint | the latest snapshot number, its hash, and where it's stored |

The program enforces the rules that matter: one account per wallet, one account per handle, handles limited to lowercase letters, digits and underscores, only the owner can add or revoke keys, and each batch's sequence number must be exactly one more than the last.

Brigade pays the rent and fees as payer. The user's wallet signs as owner. The backend never holds a user's key.

### Decision 3: no wallet popup per post

Asking a wallet to sign every post would kill the product. Instead, the browser generates its own Ed25519 key, and the user's wallet approves it once on-chain as an app key. After that, every post is signed by the browser key, silently.

With the embedded wallet this is automatic: when you sign in, Brigade registers your account and approves your browser key in one sponsored transaction. You never see a fee or a popup.

The signed message is a small JSON envelope:

```json
{"v":1,"net":"devnet","account":"<account address>","kind":"post","body":"gm","parent":null,"media":null,"ts":1792500000000}
```

The signature covers a fixed prefix plus the exact envelope bytes, and the message ID is the SHA-256 of those bytes. The server stores exactly what it received and never re-serializes it, so the signature always checks out against the stored copy.

### Decision 4: content expires, by design

Retention is 3 to 6 months, and it isn't enforced by a delete job. It falls out of how storage is organised.

Brigade writes to a tape on Tapedrive for 3 months, then rotates to a new one. Each tape expires 6 months after it was created. So every object lives between 3 and 6 months, and expiry is the storage layer's job, not mine.

This is a deliberate product choice. The feed stays current, storage and moderation costs stay flat, and nobody's five-year-old post comes back to haunt them.

### Decision 5: batches and checkpoints instead of one transaction per post

Committing every post as its own transaction would be slow and expensive. Instead, an archiver collects posts and commits them in batches:

- **Every few seconds** it builds a Merkle tree over the new message IDs, writes a manifest listing them to Tapedrive, and commits the root and count to Solana.
- **Every hour** it writes a checkpoint snapshot to Tapedrive and records its hash on Solana. The snapshot lists accounts, active keys, every committed batch, and where every live message is stored.

The checkpoint exists so a new reader doesn't have to replay the whole log from the start. It fetches the latest checkpoint, checks the hash against Solana, and replays only the batches after it.

## Architecture

```
            browser
   (embedded wallet + app key)
               │  signed message
               ▼
          Brigade API ───────────────▶ Postgres
   (verify signature + key)       (fast index for feeds)
               │
               ▼
            archiver
     │                    │
     │ messages, media,   │ batch roots (every few seconds)
     │ manifests,         │ checkpoint hash (hourly)
     │ checkpoints        │
     ▼                    ▼
 Tapedrive            Solana: brigade-chain
 (expires in        (accounts, handles, app keys,
  3–6 months)        batch log, checkpoints)
```

Solana orders things and Tapedrive stores things. Brigade runs no consensus of its own. Because the log is keyed by its authority, anyone could run their own relay and log, and readers choose which one to trust.

## What happens when you post

1. **Sign in.** You sign in and get an embedded Solana wallet. On first sign-in, Brigade builds a transaction that registers your account and handle and approves your browser's app key. Brigade pays the fees, and the embedded wallet signs it as the owner without a prompt.
2. **Write a post.** The browser builds the envelope and signs it with the app key. No popup.
3. **The API checks it.** It verifies the signature and checks that the key is approved for your account on-chain, then indexes the post in Postgres so it shows up in feeds right away.
4. **The archiver stores it.** The signed message goes to Tapedrive as its own object, and any image goes next to it, addressed by its hash.
5. **The batch lands on Solana.** Within a few seconds the post's ID is part of a Merkle root committed to the batch log.
6. **The links appear.** Each archived post links to its Tapedrive object and to the Solana transaction that committed it, and your profile links to your on-chain account. Anyone can follow those links and check the post without trusting Brigade.

## What isn't done yet

These are in progress, not shipped:

- **Rebuilding from the chain alone.** Filling an empty Postgres purely from Solana and Tapedrive, verifying every signature and Merkle root. Key validity has to come from history, not current state, because a revoked key's account can be recreated later. Until this rebuild passes, Postgres is still the practical source of truth.
- **Signed deletes and moderation takedowns.** A delete is a signed message that removes the content from Tapedrive. The hash on Solana stays, so the record shows that something existed, but not what it said.
- **Governance articles.** Long-form posts that mirror Solana governance proposals (SIMDs), with sentiment from verified validators.
- **Receiving tokens on profiles.**
- **A developer API**, where third-party apps get delegated keys the same way the browser does.
- **Operational hardening** of the archiver and relay.
- **Mainnet.** That waits for Tapedrive to launch on mainnet.

## Takeaway

The interesting decision was where each kind of data belongs. Identity and ordering need to be verifiable and permanent, so they go on Solana. Content is big and should fade, so it goes on Tapedrive with a built-in expiry. Everything fast and queryable is an index that should be rebuildable from the other two. The pieces for that are running on devnet today. Proving the rebuild end to end is the next milestone.
