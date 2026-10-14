---
title: "Pillar: An Agent and Controller for Running Solana Validators"
date: 2026-10-14T10:00:00+05:30
description: "How I built Pillar, an open-source Rust agent and controller that supervises, heals, provisions and monitors Solana validators."
tags: ["solana", "rust", "validators", "projects"]
type: post
weight: 25
showTableOfContents: true
---

## The problem

Running one Solana validator is a lot of work. Running several is worse.

Validators crash, fall behind the cluster, or sit in bootstrap for an hour while you guess whether they're stuck or just slow. Recovering from a snapshot by hand takes several steps, and it's easy to get one wrong when you're tired. Observability is journald, a Prometheus exporter and some RPC polling, set up separately on each box. Provisioning and upgrades are a different SSH session every time.

I wanted one system where each validator looks after itself and I can watch and steer the whole fleet from a single place. That's [Pillar](https://github.com/niks3089/pillar). It's open source under Apache 2.0, and my controller UI runs at [pillar.niks3089.com](https://pillar.niks3089.com).

## What it does

Pillar is two Rust binaries:

- **Agent** (`pillar-agent`): one per validator host. It supervises the validator, checks its health, restarts or recovers it, collects system metrics, tails its logs, and runs commands the controller sends.
- **Controller** (`pillar-controller`): one per fleet. It receives status from every agent, stores history in SQLite, serves a web UI and a Prometheus `/metrics` endpoint, and pushes commands back out: provision, upgrade, restart, recover, stop.

From the UI you can onboard a host with one generated command, provision a validator from a form, stream live logs, and open Grafana dashboards per node or for the whole fleet. The controller installer sets up Prometheus and Grafana with the dashboards already in place.

## Architecture

```
  Validator host (one per node)                    Controller (one per fleet)
 ┌──────────────────────────────────┐            ┌─────────────────────────────┐
 │ pillar-agent (one tokio runtime) │            │ pillar-controller           │
 │                                  │  gRPC/TLS  │                             │
 │  Reconciler ◄── mpsc ── gRPC ────┼───────────►│ gRPC server (5 RPCs)        │
 │      │                client     │  (agent    │   │            │            │
 │      ▼                           │   dials    │   ▼            ▼            │
 │  Arc<RwLock<NodeStatus>>         │   out)     │ NodeRegistry  SQLite        │
 │      ▲        │                  │            │   │  (nodes, history, logs) │
 │  Metrics      ▼                  │            │   ▼            │            │
 │  updater   HTTP /health /metrics │            │ JSON API + SSE + web UI     │
 │                                  │            │ /metrics ──► Prometheus     │
 │  Log collector (journalctl) ─────┼───────────►│              └► Grafana     │
 │      │                           │            └─────────────────────────────┘
 │      ▼ systemctl / scripts       │
 │  validator (systemd unit)        │
 └──────────────────────────────────┘
```

Both binaries depend on a small `pillar-shared` crate that holds the protobuf definitions. The gRPC service has five RPCs: `RegisterNode`, `ReportStatus`, `CommandStream`, `ReportScriptResult` and `PushLogs`. That proto file is the only contract between the two programs.

One message type, `NodeStatus`, runs through the whole system. The reconciler writes it, the metrics updater adds system metrics, the gRPC client reports it every 10 seconds, and the controller stores it and serves it to the UI and `/metrics`. Both crates use prost's `extern_path` so their gRPC stubs point at the shared types. There is one Rust `NodeStatus`, not two generated copies that drift apart.

Agents always dial out to the controller. Validator hosts never need an inbound management port. If the controller goes away, the agent retries with exponential backoff from 1 second up to 60, and keeps supervising the validator in the meantime.

## The reconcile loop

The agent's core is a reconcile loop. By default it runs every 20 seconds, and it wakes up straight away when a command arrives on its channel.

Health checks are plain JSON-RPC over `reqwest`. I deliberately left out the Solana SDK. The agent calls `getSlot` on the local node and on a reference RPC, and works out how many slots behind it is. For the validator role it also calls `getVoteAccounts`, so a node that has caught up but isn't voting counts as `behind`, not `healthy`. The states are `off`, `starting_up`, `behind` and `healthy`. A single failed check doesn't flip a node to `off`. That takes three in a row by default.

When a node is `off`, the agent tries to recover it, but most of the logic is about when *not* to:

- A provisioning script is running, or the service isn't installed yet: wait.
- The systemd service is **active**: assume it's bootstrapping. During snapshot download the RPC doesn't answer, so the node looks `off` while it's making progress, and wiping it would make things worse. The agent marks it `starting_up` and relies on the startup (1 hour) and catch-up (2 hours) timeouts to catch a node that is really stuck.
- The local major version differs from the cluster's: flag that an upgrade is needed. Restarting the wrong binary won't help.
- 3 restarts within an hour: it's a crash loop, so back off.

Only then does it act. If the local snapshot is more than 1000 slots behind, it stops the service, wipes the ledger, accounts and snapshots, tries a TCP snapshot download if one is configured (otherwise the validator bootstraps over gossip), and starts the service again. If the data is fresh, it just starts the service.

## Every command is a shell script

This is the design choice with the biggest consequences. The agent's command enum has exactly one variant: `ExecuteScript`. Provision, upgrade, restart, recover and stop are all shell script templates in `controller/scripts/`, compiled into the controller with `include_str!` and filled in with a simple `{{placeholder}}` renderer.

The agent writes the script to a temp file and runs it with `bash` as the unprivileged `sol` user in its own session, with a timeout (one hour by default). It captures stdout and stderr and sends the result back with `ReportScriptResult`. Privileged steps call `sudo` for individual commands, limited by a fixed sudoers allow-list.

Why not Rust code per client? Clients install very differently. Agave v2.x ships `agave-validator` in its tarball, but v3.x doesn't, so Pillar builds it from source. Jito is a source build too. Firedancer needs `fdctl` built from source, hugepages, and a unit that runs as root so `fdctl` can set up its PID namespace before dropping privileges. Shell fits all of that, and operators can read the exact script before it runs on a box that holds validator keys.

The cost: rendering operator input into a script that runs sudo is an injection risk. So the controller validates every field first. Client and cluster come from a fixed list, the version must match `[A-Za-z0-9._-]`, SHA-256 values must be 64 hex characters, and any path, flag or environment variable containing shell metacharacters or control characters is rejected.

## Security choices

The installer turns on TLS for gRPC. The controller generates its own CA, and agents pin `ca.pem` at install. mTLS is optional: an admin-only endpoint issues a client certificate with `CN = node_id`. The agent enrollment token and the admin API token are separate secrets, compared in constant time. Passwords are argon2-hashed. Validator keypairs never leave the host.

Some gaps are still open, and the operations guide lists them: agents share one enrollment token by default, the UI is plain HTTP unless you put a proxy in front, and `/metrics` is unauthenticated.

## Other tradeoffs

**One binary per node.** The agent started as two processes, an operator and a "link" sidecar, talking through a state file, a command file and an inotify watcher, with up to 20 seconds of polling delay. They always shipped together, so I merged them. State became an `Arc<RwLock<NodeStatus>>` and commands an in-process `mpsc` channel.

**No persistent command queue.** If the gRPC stream drops, an in-flight command is lost. Every command is operator-triggered, so the operator sees it fail and can retry.

**Parsing logs for progress.** The log collector runs `journalctl -f --output=json` per unit and picks out the validator's "downloaded N bytes X%" lines. Those numbers go into `NodeStatus`, Prometheus and a Grafana panel, so a long silent bootstrap becomes a progress bar.

## Client support

Agave is the main path and has been provisioned end to end on testnet through the UI. Jito has too, with cluster-aware MEV defaults. Writing those turned up a bug where the mainnet tip-distribution program was hardcoded for every cluster. Firedancer and Frankendancer provision and boot, but a tile still exits during init on my test host, so they aren't finished. Surfpool, a local test validator, is healthy immediately, which helps for demos and for trying an upgrade first.

## What's next

- A static or musl controller build. Right now the controller needs GLIBC 2.39, which means Ubuntu 24.04 or newer.
- Controller-side artifact storage with SHA-256 verification, so nodes don't each pull from upstream.
- A version fetcher that fills the version dropdowns from GitHub Releases.
- An `fdctl`-aware health probe, and finishing the Firedancer bring-up.
- Per-node credentials by default instead of one shared enrollment token.

If you run validators and any of this sounds useful, or wrong, the code is at [github.com/niks3089/pillar](https://github.com/niks3089/pillar). Issues and PRs are welcome.
