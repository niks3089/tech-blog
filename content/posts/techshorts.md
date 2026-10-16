---
title: "TechShorts: Designing a News Product With No Servers to Run"
date: 2026-10-14T10:00:00+05:30
description: "The constraints, decisions and tradeoffs behind rebuilding TechShorts: precompute everything, keep one source of truth, and treat the LLM as an untrusted dependency."
tags: ["architecture", "system-design", "projects"]
type: post
weight: 25
showTableOfContents: true
---

## The problem

Most tech news is the same story told five times. A launch gets covered by every major outlet, Hacker News links one of them, and you read the same facts again and again. I wanted one card per story, about 60 words, with a link out if I want the full piece.

I first built TechShorts in 2020. This rebuild is live at [techshorts.niks3089.com](https://techshorts.niks3089.com). It pulls from 27 sources every 20 minutes, collapses duplicates, writes a short summary for each story, and publishes feeds across seven topics: AI, crypto, startups, gadgets, dev, security and big tech. The most important stories also go to a Telegram channel and out as push notifications.

The interesting part isn't the feature list. It's the constraints I designed under, and what they ruled out.

## Constraints first

I set three rules before writing any code:

1. **Nothing to operate.** One person runs this in spare time. Nothing should need patching, scaling, or waking me up.
2. **Bounded cost.** The only real variable cost is the LLM. It needs a hard ceiling that doesn't depend on how busy the news is.
3. **Correct under failure.** Jobs get killed, writes fail and runs overlap. The system has to converge to the right state anyway, without me cleaning up.

Most of the design follows from taking these seriously.

## Decision 1: precompute everything, serve static files

The key observation: every reader of a topic sees the same feed. There's nothing per-user to compute at request time.

So there's no API. A scheduled batch job does all the work and writes the finished feeds as public files in object storage, with a short cache lifetime. The app reads those files directly. Personalisation ("My Feed") is just the combined feed filtered on the device against the topics you picked.

```
scheduler (every 20 min)
      │
      ▼
batch job:  fetch → dedupe → summarise → commit state → publish feeds → notify
      │
      ▼
object storage (public feeds + daily archives)
      │
      ▼
app (iOS, Android, web) reads files directly
```

What this buys: no servers to scale, no cold starts, no database, nothing in the request path that can fall over. Read traffic is the storage provider's problem.

What it costs: no per-user ranking, no search, and up to 20 minutes of staleness. For a news digest, all three are fine today. Each has a clear trigger for revisiting. When I want ranking or search, that's when an API earns its place, and not before.

## Decision 2: one source of truth, no database

All state lives in a single file in object storage. That sounds fragile, so concurrency has two layers:

- **A lease lock** so only one run executes at a time. The lease outlives the job's hard timeout, so a lock older than that always belongs to a dead run and can be taken over safely.
- **Conditional writes.** State is only written if nobody else changed it since it was read. A run working from stale state fails instead of overwriting newer data.

The lock handles the normal case and the conditional write catches anything the lock misses. Together they replace a database for a workload with one writer every 20 minutes.

## Decision 3: get the ordering right, and failures fix themselves

Within each run the order is deliberate:

1. Commit state.
2. Publish feeds and archives.
3. Send Telegram posts and push notifications.

A story counts as "new" in exactly one committed run, so it's announced at most once, even if a run dies halfway. Feeds are rebuilt from state on every run, so a failed publish repairs itself on the next one. Each stage has a time budget, so slow summarisation can't push the job past its timeout. Whatever doesn't finish is picked up next run.

Story IDs are derived from a normalised URL, with tracking parameters stripped, using a hash that's stable across builds. An ID that changed on redeploy would quietly bring back every old story. That's the kind of bug no test catches until production.

## Decision 4: treat the LLM as an untrusted dependency

The model writes the headline, the summary, the topic tags and an importance score. It also flags stories to skip: deals, sponsored posts, podcasts, roundups and opinion pieces.

I don't trust the prompt to do any of that reliably, so the model is treated like any other external input:

- **Structured output with a schema**, with topics constrained to a fixed list.
- **Validation in code.** Headlines and summaries outside the length bounds are rejected, not truncated.
- **Rejected stories are remembered for 36 hours**, so the system doesn't pay to reject the same article again on the next run.
- **Retries with backoff** on rate limits and server errors, then give up and try next run.
- **No API keys.** The job authenticates as its own service identity.

Cost is capped structurally rather than by monitoring: at most 6 new stories per source per run, and 80 per run in total. Whatever the news cycle does, the ceiling holds.

Market data doesn't go through the LLM at all. The crypto movers card is filled from a template, because numbers are exactly where a model is most likely to be confidently wrong.

## Decision 5: fetching the open internet safely

Many feeds only include a teaser, so for short items the job fetches the article page itself. That means a third party decides which URLs my job requests, which is a textbook SSRF setup.

Fetching for untrusted URLs uses a separate client whose DNS resolution refuses private, loopback, link-local and carrier-grade NAT addresses, including after redirects. Responses are size-capped. A malicious feed can't point the job at the cloud metadata service or anything internal.

## Deduplication: good enough, with a known upgrade

Duplicates are caught in two passes: same normalised URL, or titles with high word overlap within a 36-hour window. When two sources cover the same story, the higher-priority source wins.

Word overlap misses headlines that have been heavily reworded. I know that, and I chose not to start with embeddings. The upgrade path is clear: if duplicates start reaching readers, switch to semantic similarity. Until then, the simpler approach is cheaper and easy to reason about.

## Tuning the signal

The first Telegram threshold was importance 7. That came to roughly 240 posts a day, which is spam. Raising it to 9 brought it to about seven a day. Push notifications use the same bar, with a hard cap of three a day.

The lesson is a general one: any threshold on model-generated scores has to be tuned against real output volume, not chosen up front.

## What I'd revisit, and when

| Trigger | Change |
|---|---|
| Need per-user ranking or search | Add a small API over the same state |
| Duplicates reaching readers | Semantic similarity instead of word overlap |
| More than one writer, or much higher frequency | Move state into a real database |
| App ships to the stores | Real ad inventory and sponsored cards |
| Manual deploys become a bottleneck | Deploy on merge |

None of these are built, and that's the point. Each one waits for a real signal.

## Takeaway

Most of the engineering here went into deciding what not to build. Precompute when every reader sees the same thing. Let ordering and idempotency do the work a coordinator would otherwise do. Treat the LLM as untrusted input, and cap its cost by design, not by alerts. The result is a news product that runs every 20 minutes and has nothing for me to operate.
