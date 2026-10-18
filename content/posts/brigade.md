---
title: "Building Brigade: A Social Feed for Solana Where Posts Expire"
date: 2026-04-21T10:00:00+05:30
description: "How I'm building Brigade, a Solana-native social network, as a Rust modular monolith with PostgreSQL, a transactional outbox and NATS JetStream."
tags: ["solana", "social", "rust", "projects"]
type: post
weight: 25
showTableOfContents: true
---

## The problem

Most crypto discussion happens on X, and X is cut off from the things that discussion leads to. Someone posts about a token. To act on it you leave the conversation, find the right asset (hopefully not the impersonator with the same ticker), make sure the link isn't a drainer, reconnect a wallet, and then come back.

X also has no idea of wallet identity, token-gated communities or on-chain rewards. Farcaster showed a third-party app ecosystem can work, but its protocol and separate social graph add onboarding friction a first product doesn't need.

So I started [Brigade](https://brigade.niks3089.com): a public social network for the Solana ecosystem. The conversation comes first, and the crypto actions sit right next to it. The user signs every action in their own wallet.

## What it does today

Brigade is an early proof of concept. Here is what works right now:

- **Sign-in** with Google or Sign In With Solana. Users without a wallet get a user-controlled embedded Solana wallet. Posting requires a verified Gmail address; a wallet-only identity is read-only.
- **Gated signup.** You need an invitation and a Cloudflare Turnstile check before an account is created.
- **Posts, replies, threads, follows**, profiles with avatars, follower and following pages, and a following-scoped home feed with a "Latest" toggle.
- **Image attachments.** Images and GIFs only, 2 MB cap, no video.
- **30-day expiry.** Every post expires 30 days after creation unless it is preserved.
- **Cashtag price chips.** A `$TICKER` in a post renders a live price chip, but only if the ticker maps to a mint on a small verified allowlist.
- **External wallet linking** through a signed, single-use challenge.
- **A Jupiter launch endpoint.** It returns a validated external Jupiter link for a mint. It never builds, signs or submits a transaction.

## What is planned, and what is gated

The full product has a lot more in it, and I want to be precise about it. Trading, prediction markets and token distributions are regulated activities. None of them launch until partner approval and legal review are done in each jurisdiction Brigade serves. Code being finished is not the same as a feature being allowed to ship.

Here is what's planned, roughly in the roadmap's order:

- Paid "pin" and "preserve" so a post survives past 30 days, plus trade-receipt posts that show a VERIFIED stamp once the backend has checked the transaction signature on-chain.
- Solana Actions/Blinks inside posts, and sandboxed third-party mini-apps.
- Gated money features: embedded swaps, tips in approved assets, token distributions through an audited Merkle distributor program, and x402 paid resources.
- Later and gated: creator subscriptions, read-only prediction-market cards, and play-money prediction threads until the regulatory gates clear.

The navigation only shows features that work. Nothing in the UI is a disabled button promising something that isn't there.

## Architecture

The backend is a Rust modular monolith. That means one workspace, one domain model and one canonical database. The plan is to run it as separate processes (API, realtime, outbox relay, workers, Solana indexer, scheduler), but there are no network calls between made-up microservices.

```text
  Browser (Next.js on Vercel)
        |  REST /v1, cookie session
        v
  +-----------------------------+        +----------------------+
  | Rust API (Axum + Tokio)     |------->| Solana adapter       |
  |  identity, posts, graph,    |        |  RPC / DAS, behind   |
  |  feed, wallets, trading     |        |  a narrow interface  |
  +-----------------------------+        +----------------------+
     |  one transaction:        \
     |  domain row + outbox row  \--> Redis (rate limits, caches;
     v                                 disposable)
  +-----------------------------+
  | PostgreSQL (canonical)      |
  +-----------------------------+
     |  relay claims due rows
     |  (FOR UPDATE SKIP LOCKED)
     v
  NATS JetStream  --> workers: feed fanout, notifications,
                      moderation (planned)
```

The pieces:

- **PostgreSQL** is the source of truth for users, handles, wallets, posts, follows, invitations and the outbox. All access goes through SQLx with explicit SQL, and there's no ORM. I'd rather see the query plans for feed and graph queries than fight an abstraction over them.
- **Redis** holds rate limits and short-lived caches. Nothing in it is canonical, so losing it means rebuilding, not losing data.
- **NATS JetStream** is the only message broker. I'm not running Kafka next to it or using a database polling queue.
- **The frontend** is Next.js with TypeScript on Vercel. Vercel only serves presentation code. It's not the system of record and doesn't run workers.

## Decisions and tradeoffs

### A transactional outbox instead of dual writes

When a post, follow or account is created, the domain row and an outbox event are written in the same PostgreSQL transaction. A relay loop claims due rows with `FOR UPDATE SKIP LOCKED` and publishes each one to JetStream, using the event's stable ID as the NATS message ID so JetStream can drop duplicates. A row is marked published only after JetStream acknowledges it. Failures back off up to a bounded delay. Subjects are versioned, like `brigade.events.content.post-created.v1`.

The outbox is a bridge, not a second queue. Workers only consume from NATS. If NATS loses data, the canonical events can be republished from PostgreSQL. Right now the relay runs inside the API process. The feed and notification consumers on the other side are the next piece.

### Expiry as a read-time filter

The 30-day expiry is a nullable `expires_at` column with a 30-day default. `NULL` means preserved, which is the hook for paid pinning later. Every read path filters out expired posts, including the parent and root lookups in a thread, so expiry is enforced at query time and doesn't depend on a deletion job running.

An hourly task deletes media bytes for expired posts. The post rows stay because they're tiny and already invisible. This keeps storage roughly flat without a complicated cleanup pipeline.

One honest shortcut: POC images live in PostgreSQL as size-checked `BYTEA`. The architecture calls for off-host object storage behind a CDN. With a 2 MB cap and 30-day expiry, the database is fine for now. That changes before real scale.

### Fail closed on anti-abuse

Signup is the first thing a bot farm attacks, so every check fails closed:

- The Turnstile token is validated server-side against the expected hostname, action and maximum age.
- Invitations are stored hashed and consumed atomically with account creation.
- Posting limits use a Redis script with both a burst window and a sustained window.
- If Redis errors out or returns something unexpected, the post is refused, not let through.
- Quotas come from server-verified state, never from a client-supplied wallet header.

No spec can stop a determined human-assisted bot. The goal is that commodity automation can't get in, and that operators can contain abuse without a release.

### Wallet linking without replay

Linking an external wallet means signing a domain-separated message with a 32-byte random nonce. The server stores only the nonce's hash, along with the origin and an expiry. It checks all three and marks the challenge consumed. A timestamp-only scheme can be replayed inside its validity window, and a consumed nonce in the database closes that hole.

### Tokens are mints, not tickers

Token identity is always chain plus mint address. The cashtag chips only render for tickers on a verified mint allowlist, so `$USDC` can never quietly resolve to a lookalike token. Prices come from a server-side endpoint. The Solana provider's credentials never reach the browser.

### Non-custodial by construction

Brigade never holds a private key. The planned transaction flow treats every financial action as an immutable, expiring intent:

1. The provider builds the transaction.
2. Brigade decodes and allowlist-checks the programs and accounts, then simulates it.
3. The user sees a human-readable preview.
4. After signing, the signed bytes must match the previewed bytes, or the preview is invalidated.

Today's Jupiter integration stops well short of that. It hands back a link and a response that explicitly says the user's wallet has to approve and that Brigade holds no funds.

### Don't build distributed systems early

X's Manhattan and Meta's TAO exist because of scale and team sizes Brigade doesn't have. So the feed starts as PostgreSQL keyset pagination on `(created_at, id)` with an opaque cursor. Read replicas, partitioning, ClickHouse, dedicated search and hybrid fanout each have an explicit trigger in the scaling plan, and each waits until measurements say the simpler setup can't hold its targets.

## What's next

The next milestones from the roadmap:

- NATS consumers for feed projection and notifications, then a simple versioned For You ranker next to the chronological feed.
- Moderation: reports, a review console and enforcement actions with reason codes.
- Market-aware social: mint-resolved token cards with freshness labels, verified terminal deep links, and search.
- Paid preserve and verified trade receipts.
- The mini-app sandbox: a separate origin, capability grants and a kill switch.

The money features come after that. Each one ships separately and only after its own security review, partner authorization and legal sign-off.

If you want to see where it is today, the POC is live at [brigade.niks3089.com](https://brigade.niks3089.com).
