---
title: "Solana Needs a Social Layer for Governance"
date: 2026-04-21T10:00:00+05:30
description: "Why governance and community chat on Solana need a wallet-native social layer next to Realms, what that layer has to do, and how I'd architect it."
tags: ["solana", "governance", "architecture", "projects"]
type: post
weight: 25
showTableOfContents: true
---

## The problem

On Solana, the vote and the conversation about the vote happen in different places.

[Realms](https://realms.today) and the SPL Governance program cover the on-chain half well: realms, proposals, voting power, vote records, execution. But the discussion that decides how people vote happens somewhere else: a Discord server, a forum thread, a post on X. None of those places know who holds governance power. None of them link a message to the proposal it's about. Most of them can't tell a real token holder from someone with a matching profile picture.

The result is familiar to anyone who has sat through a DAO vote:

- **Discussion is detached from the decision.** A proposal goes live on-chain while the reasoning is scattered across three apps, and much of it is gone a month later.
- **Identity is weak.** Anyone can claim to speak for a large holder. Sybil accounts are free.
- **Participation is low.** Voting means leaving the conversation, finding the proposal, connecting a wallet and then coming back.
- **Nothing is portable.** A community's history is locked inside whichever chat platform it happened to choose.

The chain already has the hard part: verifiable identity, verifiable holdings and verifiable votes. What's missing is a social layer that uses them.

## What needs to be built

Not another chat app. A layer where the wallet is the identity, and governance state is part of the conversation.

**Wallet-native identity.** You sign in with a wallet. Your profile can show which realms you're in and how much voting power you have, proven from chain state, not claimed. Linking more wallets uses signed, single-use challenges, so a link can't be replayed.

**Realm-scoped spaces.** Each realm gets a space whose membership and roles come from governance state: community token holders, council members, delegates. Gating is computed from chain data, not from a bot assigning roles.

**Proposal-linked threads.** Every proposal gets a discussion thread by default. The thread shows live status (draft, voting, succeeded, executed), the current tally, time remaining and who has voted, all read from chain. Arguments for and against sit next to the result they affected, and they stay there.

**Voting from inside the conversation.** A vote button that builds the governance instruction, shows a human-readable preview, and hands it to your own wallet to sign. The platform never holds a key.

**Signal weighted by stake.** Optionally sort or filter discussion by verified governance power, so a holder's argument isn't buried under noise. It's a lens, not censorship: everything is still readable.

**Notifications that matter.** A proposal was created in a realm you're in. Voting ends in 24 hours. Your delegate voted. A proposal you discussed was executed.

**Moderation owned by the community.** Each realm moderates its own space, with tools that can be delegated to its council, and every enforcement action records a reason code.

## Architecture

The shape I'd use is a modular monolith: one codebase, one domain model and one canonical database, deployed as a few separate processes. Splitting it into services early would buy coordination problems before it buys scale.

```
                Solana (SPL Governance, token accounts)
                      │ account updates
                      ▼
            ┌───────────────────────┐
            │ governance indexer    │  realms, proposals,
            │ (stream + backfill)   │  vote records, voting power
            └──────────┬────────────┘
                       │ one transaction: rows + outbox event
                       ▼
 web / mobile ──▶ API ──▶ PostgreSQL (canonical) ──▶ outbox relay ──▶ NATS JetStream
      ▲                                                                    │
      │ realtime (websocket)                                               ▼
      └──────────── realtime gateway ◀── workers: feed fanout, notifications,
                                          thread creation, moderation
```

**PostgreSQL is the source of truth** for users, wallets, spaces, posts and the indexed governance state. Feeds and threads begin as keyset-paginated queries. Read replicas, partitioning and dedicated search each wait for a measured trigger.

**A governance indexer** follows the governance program's accounts in real time and backfills history. It turns raw accounts into realms, proposals, vote records and voting power. When it sees a new proposal it writes the proposal row and a `proposal-created` event in the same transaction, so a thread can never exist without its proposal, or the other way round.

**A transactional outbox, not dual writes.** Every domain change writes its event in the same transaction as the data. A relay publishes those events to NATS JetStream, using the event ID to drop duplicates. Workers only consume from NATS, so if the broker loses data, events can be replayed from Postgres.

**Workers turn events into side effects:** create the proposal thread, fan posts out to feeds, send notifications, and run moderation checks. Each is idempotent, so redelivery is harmless.

**A realtime gateway** pushes new messages and tally changes to connected clients. It reads from NATS and holds no state of its own, so it scales horizontally.

**A Solana adapter behind a narrow interface.** RPC, account streaming and price data all go through one boundary, so switching providers doesn't touch the domain code, and provider credentials never reach the browser.

## Non-custodial by construction

Voting, and any other on-chain action, follows one flow:

1. Build the transaction server-side from the governance instruction.
2. Decode it and check every program and account against an allowlist, then simulate it.
3. Show the user a human-readable preview: realm, proposal, choice and voting weight.
4. The user signs in their own wallet.
5. The signed bytes must match the previewed bytes, or the preview is thrown away.

The platform can propose a transaction but never sign one. That boundary matters more in governance than anywhere else, because a malicious vote, or a disguised token transfer, could decide the outcome.

## Hard problems worth naming

**Voting power isn't just token balance.** It depends on deposits, delegation, council tokens and voter-weight plugins. The indexer has to compute it the way the governance program does, or the UI will show numbers that don't match the vote.

**Sybil resistance has limits.** Wallets are free too. Weighting by governance power helps, and so do gated signup and rate limits, but none of it stops a determined holder from splitting tokens. The goal is to make cheap manipulation expensive, not to claim it's impossible.

**Chain reorgs and lag.** Tallies shown in a thread must say how fresh they are, and the indexer must reconcile against finalized state.

**Privacy.** Linking wallets to a social profile reveals holdings. Linking has to be opt-in for every wallet, and the default view should show governance roles, not balances.

**Moderation versus censorship-resistance.** Communities need moderation, but nobody wants a platform that can quietly remove dissent before a vote. Recording reason codes and keeping governance-related moderation visible in public logs helps keep both.

## Where to start

The order I'd build it in:

1. Wallet sign-in, realm-scoped spaces and proposal-linked threads with live status, read-only from chain.
2. Voting from the thread through the non-custodial flow.
3. Notifications and delegate activity.
4. Stake-weighted views and moderation tools for each realm.
5. Opening the platform to third-party mini-apps for treasury dashboards, grants and other realm-specific tools.

Each step is useful by itself, and none of them needs the next one to justify it.

I've been prototyping parts of this as [Brigade](https://brigade.niks3089.com). The main argument stands without it, though. Solana already has verifiable identity, holdings and votes. Governance deserves a conversation layer that actually uses them.
