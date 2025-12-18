# PHOBOS — High‑Performance On‑Chain Price Graph & MEV Research (Osmosis)

> **Scope:** This repository documents a multi‑month research and engineering effort to design, build, and operate a *low‑latency, on‑chain price graph* and MEV experimentation stack on **Osmosis**.  
> **Focus:** price discovery quality, latency, correctness under real chain conditions, and scalable ingestion — not production trading claims.

---

## Executive Summary

PHOBOS is a self‑hosted, locally‑validated market‑data and execution research platform designed to answer one core question:

> *Can we construct a high‑fidelity, real‑time price graph from raw on‑chain state that is fast enough and accurate enough to support advanced arbitrage and MEV strategies?*

The answer was **yes — from a systems and data‑quality perspective**.

Key outcomes:
- Built a **real‑time price graph** spanning **CL (Concentrated Liquidity)** and **GAMM (Balancer)** pools
- Achieved **sub‑200 ms GAMM refresh cycles** and **~500–600 ms CL refresh cycles** on a local node
- Implemented **sigma‑aware pricing**, oracle gating, and liquidity‑weighted edges
- Proved correctness via **truth harnesses** against live chain state
- Designed the system to scale horizontally and port to **Solana / Jito**‑style environments

This repo intentionally emphasizes **architecture, data quality, and performance**, not financial results.

---

## Why Osmosis?

Osmosis offered a uniquely demanding environment:
- Two fundamentally different AMM models (GAMM vs CL)
- High on‑chain activity and arbitrage competition
- Rich gRPC + LCD surface area
- Complex tick‑based liquidity math

If a price graph can remain correct and performant here, it can be adapted anywhere.

---

## System Architecture (High Level)

```
                ┌────────────────────────┐
                │  Local Osmosis Node    │
                │  (validator / full)   │
                └──────────┬────────────┘
                           │
                ┌──────────▼────────────┐
                │  Price Graph Daemon   │
                │  (Rust, async)        │
                │                        │
                │  • CL live scanner     │
                │  • GAMM fast poller    │
                │  • Sigma + oracle      │
                │  • Liquidity math      │
                └──────────┬────────────┘
                           │ gRPC
                ┌──────────▼────────────┐
                │   In‑proc Clients     │
                │  • Arbitrage engine   │
                │  • Front‑run engine   │
                │  • Metrics / debug    │
                └───────────────────────┘
```

All market data is derived **directly from the local node** — no third‑party RPCs.

---

## Price Graph Design

### Core Concepts

Each edge in the graph represents a *directional executable price*:

```text
(pool_id, denom_in → denom_out)
  mid_price
  fair_price (oracle)
  sigma (volatility)
  virtual_liquidity (USD)
  pool_type (CL | GAMM)
```

The graph is **directional** and **pool‑aware** — edges are never blended across pools.

---

## CL (Concentrated Liquidity) Pipeline

### Live CL Scanner

- Pulls pool state via **Osmosis gRPC**
- Reconstructs:
  - current tick
  - tick spacing
  - prefix liquidity
  - net liquidity above / below tick

### Key Challenges Solved

- Correctly parsing **sdk.Dec** values (including scientific notation)
- Avoiding integer overflow while preserving precision
- Handling asymmetric liquidity nets
- Matching gRPC math with on‑chain truth

### Performance

- ~500–600 ms average full‑list scan
- gRPC channel pooling + parallel net queries
- Cycle latency exposed via Prometheus histogram:

```
cl_fast_cycle_seconds_bucket
cl_fast_cycle_seconds_sum
cl_fast_cycle_seconds_count
```

---

## GAMM (Balancer) Pipeline

### Fast List‑Based Poller

- Polls only a **curated pool list** (TVL‑filtered)
- Interval‑based (milliseconds)
- Concurrency‑bounded
- Reloads pool list on file change

### Performance

- ~130 ms average cycle across ~90 pools
- Stable under sustained load
- Metric:

```
gamm_fast_cycle_seconds_bucket
```

---

## Oracle & Sigma Handling

Each edge can be:
- **Oracle‑backed** (fair_price + sigma)
- **Oracle‑optional** (mid‑only, gated)

### Sigma Usage

- Sigma used for:
  - risk‑aware sizing
  - edge scoring
  - bait detection (front‑run path)

- Zero‑sigma edges are explicitly handled (floored, never divided by zero)

---

## Historical Evolution

### Phase 1 — Tickmap Discovery

- Three‑tier tickmap system:
  1. bitmap locator
  2. master index
  3. trimmed execution map

Goal: correctness and full coverage.

### Phase 2 — Truth Harness

- Side‑by‑side comparison of:
  - reconstructed reserves
  - implied prices
  - on‑chain execution results

This phase eliminated multiple silent math bugs.

### Phase 3 — Live Scanners

- Replaced file‑based watchers with **live gRPC scanners**
- Introduced:
  - channel pools
  - parallel direction scans
  - deterministic cycle timing

### Phase 4 — Unified Price Graph

- Single gRPC service exporting:
  - ListEdges
  - Direct price queries
  - Pool‑scoped snapshots

---

## Arbitrage Engine (Research)

- Linear USD‑anchored cycles (USDC → X → USDC)
- Pool‑aware (never mixes pools incorrectly)
- Slippage + impact‑aware sizing
- Wallet‑safe routing

> This engine was used primarily as a **data‑quality validator**.

---

## Front‑Run Engine (Research)

- Mempool ingestion via Tendermint WebSocket
- Pool‑specific edge lookup (never global quotes)
- Pre‑ask revalidation
- Oracle‑aware bait veto

Again: **focus was correctness and timing**, not production claims.

---

## Observability

Everything is measurable:

- Refresh latency (CL & GAMM)
- Edge counts
- Oracle freshness
- Cache age
- Execution veto reasons

Prometheus endpoint:
```
http://localhost:9898/metrics
```

---

## Why This Matters for Jito / Solana

PHOBOS demonstrates:

- Building **low‑latency market structure** from raw chain state
- Designing **pool‑aware, directional price graphs**
- Handling **high‑frequency data correctness** under adversarial conditions
- Architecting systems that cleanly port to:
  - Solana validators
  - Shredstream / block‑engine environments

The Osmosis work serves as a **laboratory** — not an endpoint.

---

## What This Repo Is (and Is Not)

✅ Systems engineering
✅ Market‑data correctness
✅ Performance instrumentation

❌ Marketing claims
❌ Profit screenshots
❌ Turn‑key trading advice

---

## Closing

PHOBOS is best understood as a **market‑data and MEV research platform** that happens to include arbitrage and front‑run engines as *validation tools*.

The price graph — its latency, correctness, and observability — is the real artifact.

---

*Happy to discuss architecture, performance trade‑offs, or how this maps directly onto Jito / Solana block‑engine workflows.*

# solana
Phobos-Solana Project
