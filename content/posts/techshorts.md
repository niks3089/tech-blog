---
title: "Rebuilding TechShorts: tech news in 60 words, with Rust and static JSON"
date: 2026-10-14T13:00:00+05:30
description: "How I rebuilt my 2020 hobby app as a Rust ingest job that turns 27 tech feeds into 60-word cards, served as static JSON."
tags: ["rust", "news", "gcp", "projects"]
type: post
weight: 25
showTableOfContents: true
---

## The problem

I read a lot of tech news, and most of it is the same story told five times. TechCrunch, The Verge and Ars Technica all cover the same launch. Hacker News links one of them. I wanted one card per story, about 60 words long, with a link to the source if I want more.

I built an app called TechShorts in 2020 as a hobby project. This is the rebuild. It's live at [techshorts.niks3089.com](https://techshorts.niks3089.com).

Every story is a card: a short headline, a summary of at most 60 words, a source and a link. You swipe through them one at a time. Cards are tagged with seven topics: AI, crypto, startups, gadgets, dev, security and big tech.

## What it does

A Rust job runs every 20 minutes. It pulls RSS and Atom feeds from 27 sources, drops duplicates, has an LLM write the cards, and publishes the results as static JSON files. Each topic gets its own feed, plus one combined feed. The most important stories also go to a Telegram channel and out as push notifications.

The frontend is an Expo (React Native) app that reads those JSON files. The same codebase builds for iOS, Android and the web.

## Architecture

```
Cloud Scheduler (every 20 min)
        │
        ▼
Cloud Run Job: techshorts ingest (one Rust binary)
  1. lock      lease lock object in GCS, one run at a time
  2. fetch     27 RSS/Atom feeds from sources.toml, 16 in parallel
  3. dedupe    canonical URL hash + title similarity over the last 36h
  4. enrich    fetch the article page if the feed text is too short
  5. summarize Gemini 2.5 Flash on Vertex AI, structured JSON output
     market    CoinGecko movers and trending coins, templated, no LLM
  6. store     state/items.json in GCS, generation-matched write
  7. publish   feed/all.json, feed/{topic}.json, archive/YYYY-MM-DD.json
  8. notify    FCM push (top stories) and Telegram (importance >= 9)
        │
        ▼
Expo app (iOS, Android, web) reads the static JSON directly
```

There's no API server. The feed is the same for everyone who reads a given topic, so it can be a public object in a storage bucket with a five-minute cache header. Nothing to cold-start and nothing to scale. I'll add a real API when I need per-user ranking or search, and not before.

## Sources

Sources live in `sources.toml`. Each one is a name, a feed URL, a default topic and a priority:

```toml
[[source]]
name = "CoinDesk"
url = "https://www.coindesk.com/arc/outboundfeeds/rss/"
topic = "crypto"
priority = 9
```

Adding a source means adding one block. The list mixes general tech sites, AI lab blogs, crypto outlets, Indian startup news, dev sources (including Hacker News front page posts with 200+ points), security and gadget sites.

The topic is only a hint for the summarizer, which can retag a story. The priority decides who wins when two sources cover the same story.

## Dedupe

Dedupe happens in two passes.

First, URLs are canonicalized. I strip `utm_*`, `ref`, `fbclid`, `gclid` and similar tracking params, drop the fragment and the trailing slash, then hash the result with FNV-1a to get the card id. I picked FNV-1a over Rust's `DefaultHasher` on purpose: `DefaultHasher` isn't guaranteed stable across builds, and an id that changes on redeploy would bring back every old story.

Second, titles are compared. Each title becomes a set of lowercase words of three or more letters, with stopwords removed. Two titles count as the same story if their Dice coefficient is 0.6 or higher. New candidates are compared against each other and against every card from the last 36 hours. Candidates are sorted by source priority before this pass, so the higher-priority source claims the story.

Word overlap misses rephrased headlines. If duplicates start slipping through, the upgrade is embeddings and cosine similarity.

Two caps sit on top: 6 new items per source per run, so one busy feed can't flood the queue, and 80 per run in total, which caps LLM spend.

## Summarisation

Many feeds only carry a teaser. If a candidate has less than 400 characters of text, the job fetches the article page and pulls out the paragraph text. That fetch uses a separate HTTP client for untrusted URLs. Its DNS resolver drops private, loopback, link-local and CGNAT addresses, including on redirects, so a feed can't point the job at the cloud metadata server or anything internal. Responses are capped at 5 MB for feeds and 2 MB for pages.

The text then goes to Gemini 2.5 Flash on Vertex AI. The prompt asks for a 6 to 10 word headline, a 45 to 55 word factual summary, one or two topics, and an importance score from 1 to 10. The model also returns a `skip` flag for deals, sponsored posts, podcast episodes, newsletter roundups and opinion pieces.

I don't trust the prompt to enforce lengths. The request uses a JSON response schema with the topics as an enum, and the code checks every result: 1 to 12 words for the headline and 20 to 60 for the summary. Anything outside that is rejected. Rejected ids are remembered for 36 hours so the job doesn't pay to reject the same article again on the next run.

Auth uses the job's service account, so there are no LLM API keys. A 429 or 5xx gets retried with backoff, up to three attempts.

Crypto prices skip the LLM. A CoinGecko card with BTC and ETH prices, the biggest movers in the top 100 and trending coins is filled from a template. Its id includes the date and a 6-hour slot, so there's at most one per slot.

## Storage and scheduling

State is a single JSON file in Google Cloud Storage. No database. Every write is conditional on the object's generation number, so a run that read stale state fails instead of overwriting newer data.

On top of that, a lease lock serializes whole runs. The lock is another GCS object that holds the time it was taken. Taking it is a conditional create, or a conditional replace once the lease has expired. The lease is 16 minutes because Cloud Run kills the task at 15, so an older lock belongs to a dead run.

Cards are kept for 90 days. Each run rewrites the feeds (latest 200 per topic, 300 combined) and one archive file per UTC day. Since ingest only accepts stories from the last 36 hours, only recent archives can change, so each run rebuilds the last four. That also repairs a write that failed last time. The job deletes archives past 90 days, with a bucket lifecycle rule as a backstop.

Order matters. State is committed first, then feeds and archives, then pushes and Telegram. A card is "new" in exactly one committed run, so it's posted at most once. Summarization gets 8 minutes and Telegram 3, so neither can push the job past its timeout. Whatever didn't finish is picked up next run.

The image is a multi-stage Docker build that ships the binary on `debian:bookworm-slim` as an unprivileged user. `deploy.sh` is idempotent: IAM, image build, the Cloud Run Job and the 20-minute scheduler trigger.

## Telegram and push

Telegram posting is a `sendMessage` call from the same job, so there's no bot server. The first threshold was importance 7, which came to about 240 posts a day. That's spam, so it's now 9, roughly seven a day.

Push notifications go through FCM to a `top` topic that the app subscribes to. Same importance bar, capped at three a day. The send times are stored in state to enforce the cap.

## The app

The app is Expo and React Native, because ads and push are first-class there and not in the Rust mobile options.

On first launch you pick your interests. "My Feed" is just the combined feed filtered on the device against those interests. One shared file means there's no per-user backend. At the end of the feed, the app walks back through the daily archives, up to 90 days. Responses for a tab you've left are dropped, so a slow request can't overwrite the current screen.

On desktop browsers, a wheel or trackpad flick moved less than half a card and snapped back, so the web build turns each gesture into exactly one card.

On iOS and Android there's an ad card after every sixth story. It still uses Google's test ad units, since the AdMob account isn't set up yet.

## What's next

These are plans, not features yet:

- **Store release.** There are EAS build profiles for iOS and Android, but the app isn't on the App Store or Play Store yet.
- **Real ads and sponsored cards.** Swap the test ad units for real ones, and add direct-sold cards marked "Sponsored" to the feed JSON.
- **Topic Telegram channels.** Right now everything goes to one channel. Per-topic channels for AI and crypto are already sketched in the config.
- **Better dedupe** if word overlap stops being good enough.
- **CI deploys on merge**, replacing a manual `./deploy.sh`.

Most of the work in this rebuild went into the pipeline. Because its output is static JSON, there's still no server to run.
