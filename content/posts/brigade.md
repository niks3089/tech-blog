---
title: "Pitch: A Wallet-Native Social Network for Solana"
date: 2026-04-21T10:00:00+05:30
description: "A product pitch, not a launch: why Solana needs a social network built around wallets, what the product should be, and how governance with Realms fits in as one feature."
tags: ["pitch", "solana", "social", "product"]
type: post
weight: 25
showTableOfContents: true
---

> **This is a pitch.** Nothing described here is built, launched or in development. It's a product proposal: the problem, the product I think should exist, and how I'd approach it. "Brigade" is just a working name.

## The one-liner

A social network where your wallet is your identity. Solana communities talk, coordinate, act on chain and govern themselves in one place, without leaving the conversation.

## The problem

Solana's social life happens on platforms that know nothing about the chain.

Market talk happens on X. Community chat is on Discord and Telegram. Governance discussion is in forums, with the votes themselves on [Realms](https://realms.today). Creators get paid somewhere else again. None of these places know who holds what, who is actually a member of a community, or which token a ticker refers to.

What that costs people:

- **Context switching.** Someone posts about a token, a proposal or a mint. To act on it you leave, find the right page, reconnect a wallet and come back. Most people don't.
- **Impersonation and scams.** Fake tickers, lookalike accounts and drainer links spread in exactly the places where people make on-chain decisions.
- **Weak identity.** Anyone can claim to be a holder, a team member or a whale. Real stake has no weight in the conversation.
- **No ownership.** A community's history and relationships are locked inside platforms it doesn't control.

The chain already provides verifiable identity, holdings and actions. The social layer doesn't use any of them.

## Who it's for

- **Holders and traders** who want market conversation with verified tokens and safe links.
- **Communities and projects** that want a home where membership, roles and gating come from on-chain state.
- **Creators** who want to be rewarded directly by their audience, in crypto.
- **DAOs** that want discussion and voting in the same place as their community.
- **Developers** who want to put their app in front of an existing social audience.

## The product

### Core experience (MVP)

1. **Sign in with your wallet.** Your profile can show verified holdings, communities and roles, proven from chain state, not claimed. New users get an embedded wallet, so no crypto setup is needed to start.
2. **A public feed** for posts, replies and follows, with market conversation built in. Posts expire after 30 days unless someone pays to keep them (more on this below).
3. **Verified tokens, not tickers.** A cashtag resolves to a specific mint from a verified list, with a live price, so `$USDC` can never point to a lookalike.
4. **Communities.** Spaces for projects and groups, with membership and roles that can be gated by on-chain state, such as holding a token or an NFT.
5. **Safe links.** Links to on-chain actions are validated before they're shown, and every action is signed in the user's own wallet. The product never holds funds or keys.

### Features on top

- **Governance.** One feature among many, built with Realms. Each proposal gets a discussion thread with live status and tally, and members can vote from the thread with their own wallet.
- **Creator rewards.** Tips and rewards from followers directly to creators.
- **Actions in posts.** Solana Actions and Blinks render inline, so you can mint, donate or vote without leaving the feed.
- **Notifications** for mentions, community activity, proposals closing and actions you took part in.

### Later

- **Mini-apps.** Third-party apps that run inside communities, such as treasury dashboards, games and tools, in a sandbox with explicit permissions.
- **Token drops and distributions** for communities, through audited on-chain programs.
- **Embedded swaps and trading.** These are regulated activities, so they only ship after partner approval and legal review in each market.

## Why Solana, why now

- **Wallet onboarding has caught up.** Embedded and mobile wallets make signing in and signing a transaction quick enough for regular users.
- **Cheap, fast transactions** make on-chain actions from a feed practical. A tip or a vote is a tap, not a fee decision.
- **Standard building blocks exist:** Actions and Blinks for inline actions, Realms for governance, and token standards for gating and verification.

## How it fits with what exists

| | X | Discord / Telegram | Farcaster | This product |
|---|---|---|---|---|
| Public conversation | Yes | Limited | Yes | Yes |
| Verified holder identity | No | Via bots | Partial | Native |
| Token-gated communities | No | Via bots | Limited | Native |
| On-chain actions in context | No | No | Frames | Native, non-custodial |
| Governance with discussion | No | No | No | Yes, with Realms |

The point isn't to replace any one of these. It's a home where Solana's social activity and on-chain activity finally meet.

## Why not just build on Farcaster?

Farcaster proved that decentralized social can work, and that interactive posts drive engagement. But for Solana communities it brings friction they don't need:

- **Identity lives on another chain.** Accounts, keys and storage rights are registered on Optimism. A Solana wallet can be linked, but it's never the core identity.
- **Paying to exist.** Signing up means a yearly storage fee paid in ETH, plumbing that users resent and that Solana users have to bridge for.
- **An empty graph.** It's a separate social graph whose most active communities are in the Ethereum and Base ecosystem, not Solana.
- **Built Ethereum-first.** Frames and the in-app wallet were designed around EVM transactions, and on-chain state like token holdings doesn't natively shape channels or identity.

Here's what I'd do differently. Farcaster made decentralized social feel like infrastructure. This product should make Solana feel like a place.

1. **Posts expire, and permanence is earned.** This is the flagship inversion. Instead of storage rent nobody wants to think about, posts expire after 30 days by default and permanence becomes a social mechanic:
   - **Pin:** the author pays a little SOL to keep a post.
   - **Preserve:** anyone pays to keep someone else's post. The fee is shared with the author, and the post shows how many people preserved it, like "Preserved by 214".
   - **Mint:** the most-preserved posts can become compressed NFTs, which Solana makes almost free.
2. **Verified trade receipts.** A post can attach a transaction signature that's checked on-chain and stamped *Verified*. The entry and timing are real and the direction is provable, while position size can stay hidden. Add a verified 30-day P&L leaderboard. Crypto Twitter runs on screenshots that anyone can fake, and Farcaster has nothing equivalent.
3. **Actions and Blinks instead of Frames.** Farcaster had to invent Frames. Solana already has Actions and Blinks, so any post can carry a live price chip, a tip, a drop claim or a one-tap swap without leaving the feed. Signing always happens in the user's own wallet.
4. **Token-gated communities.** Like Farcaster channels, but with skin in the game: hold a set amount of a token to enter, enforced by a wallet balance check.
5. **Supporting mechanics:**
   - **Prediction threads,** play-money only until the regulatory gates clear.
   - **Images and GIFs only, no video.** Since everything expires anyway, this keeps storage and moderation costs flat.
   - **Invite scarcity as a growth loop,** the way early Warpcast grew.

## How we'd know it works

- **Activation:** the share of new users who post, follow or join a community in their first week.
- **Actions from conversation:** on-chain actions started from a post or thread (tips, votes, mints).
- **Community health:** weekly active members per community, and retention after 30 days.
- **Trust:** reports of scams and impersonation per thousand users, which should fall as verification improves.

## Go to market

1. **Start with a few active communities** that already feel the pain, and design the MVP with them.
2. **Lead with verified identity and safe links.** That's where existing platforms fail Solana users most visibly.
3. **Add features that bring whole communities over,** like gated spaces, governance threads and creator rewards.
4. **Open the platform** to mini-apps once there's an audience to serve.

**Business model, to be validated:** free for users and communities, with paid tools for community operators (analytics, moderation, custom branding), boosted posts that are clearly labelled, and a revenue share on mini-apps.

## Risks and open questions

- **Network effects.** People are already on X and Discord. The wedge has to be what those platforms can't do: verified on-chain identity and safe actions.
- **Regulation.** Trading, token distributions and anything resembling financial services need partner approval and legal review in each market. Those features are gated, not assumed.
- **Sybil resistance has limits.** Verification raises the cost of fake accounts, but it doesn't eliminate them.
- **Privacy.** Linking wallets to a profile reveals holdings. Linking must be opt-in for each wallet, and holdings should be hidden by default.
- **Moderation.** Open communities need moderation tools without making the platform a central gatekeeper.

## A sketch of the architecture

This is deliberately high level. At this stage the product matters more than the stack.

```
           Solana (tokens, Actions, Realms)
                     │
                     ▼
   chain indexer ──▶ database (single source of truth) ──▶ events ──▶ feeds, notifications,
         ▲                       │                                   community + governance threads
         │                       ▼
  non-custodial actions ◀── app (web + mobile, realtime)
  (preview, then sign in wallet)
```

- **Index the on-chain state the product needs** (holdings for gating, verified token mints, governance proposals) into one canonical database.
- **Drive features from events:** a new post fans out to feeds, a new proposal opens its thread, and an action triggers a notification.
- **Never custody anything.** Every action is built, checked and simulated, then shown in plain language and signed in the user's wallet. What gets signed must match what was previewed.
- **Start as one well-structured system,** and split it only when real usage says to.

## The ask

If you run a Solana community, create content in the ecosystem or build apps that could live inside a social feed, I'd like to hear whether this matches what you need, and what's missing.
