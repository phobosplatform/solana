# Solana Raydium MEV Sandwicher (Rust + Jito)

This repository contains a **research‑grade Solana MEV bot** that targets
Raydium AMM pools and constructs **front‑run / back‑run bundles** around large
swaps, with routing through the **Jito block engine**.

The design is intentionally modular:

- Pure‑Rust core (Tokio + Axum)  
- Live Raydium pool discovery & state tracking  
- On‑chain volatility (σ) & impact estimation per pool  
- Simple x·y = k sandwich simulator  
- Jito bundle construction with risk caps & accounting logs  

The code is written to be a realistic stepping‑stone toward **Shredstream‑based
atomic sandwiching**, while remaining safe to run in “research” mode today.

---

## High‑level Architecture

At a high level, the bot does:

1. **Pool discovery & tracking**
   - Fetches Raydium pool metadata from the public SDK/liquidity API.
   - Maintains a bounded map of tracked pools (e.g. up to `MAX_POOLS`).
   - Periodically refreshes vault balances via Solana RPC.
   - Computes per‑pool price, raw σ, and a rolling σ window.

2. **Opportunity detection via webhook**
   - Exposes a small Axum HTTP server that accepts **transaction webhooks**
     (Helius‑style JSON payloads).
   - Filters for Raydium swaps that match supported pools (e.g. SOL/USDC).
   - Extracts opportunity size, direction, and basic metadata.

3. **Local RPC enrichment**
   - For each candidate, the bot queries a Solana RPC node
     (typically a local validator with transaction history enabled) to:
     - Fetch the raw `VersionedTransaction`.
     - Decode instructions and accounts.
     - Ensure the opportunity can be included in a Jito bundle.

4. **Sandwich simulation & gating**
   - Uses a simple x·y = k model to simulate:
     - Our front‑run (SOL → token).
     - Opportunity trade at new reserves.
     - Our back‑run (token → SOL).
   - Computes:
     - Expected **gross profit** in SOL.
     - Expected **net profit** in SOL (after a configurable Jito tip).
     - Opportunity price impact and pool σ.
   - Applies multiple safety filters, for example:
     - Minimum opportunity size.
     - Maximum allowed price impact.
     - Maximum allowed σ.
     - Minimum **net profit** in SOL.
     - Per‑session caps on total notional and total Jito tips.

5. **Bundle construction & submission**
   - Builds three real Solana transactions:
     1. **Front‑run swap** (MEV wallet SOL/wSOL → SPL token, via Raydium).
     2. **Back‑run swap** (SPL token → SOL/wSOL).
     3. **Tip transfer** (MEV wallet → Jito tip account).
   - Optionally includes the **opportunity transaction** fetched from RPC.
   - Encodes them as a Jito bundle and POSTs to the block engine HTTP API
     when in `Live` mode.

6. **Accounting & observability**
   - Emits structured logs for:
     - Every candidate (`[CANDIDATE]`).
     - Every gating decision (`[SKIP]` with reason flags).
     - Every live bundle attempt (`[JITO LIVE ATTEMPT]`).
     - Every successful submission (`[JITO LIVE SENT]`).
     - A dedicated `[ACCT]` line with all fields needed for PnL/tax export.
   - Designed so a separate process can tail logs and compute:
     - Win rate.
     - Realized/realizable PnL in SOL and USD.
     - Tip spend vs. gross MEV.

> **Note:** This repository never commits real keys, wallet addresses, or
> infrastructure details. All sensitive configuration is injected via
> environment variables at runtime.

---

## Components

### 1. Core binary (`oracle`)

The main binary:

- Starts an Axum HTTP server to receive webhooks.
- Spawns a pool‑tracking task to keep Raydium state fresh.
- Manages shared state (`PoolState`, `PoolConfig`, Jito config + per‑session
  counters).
- Coordinates:
  - RPC client (non‑blocking Solana RPC).
  - HTTP client (Raydium API, Jito block engine).
  - Pyth/Hermes price feed for SOL/USD.

### 2. Pool & volatility tracking

For each tracked pool, the bot maintains:

- **Reserves**: base/quote vault balances.
- **Pool price** (base → quote).
- A rolling history window of price changes.
- **σ (sigma)** computed from the rolling window and stored per pool.

This allows it to:

- Ignore illiquid or “dead” pools.
- Prefer pools whose volatility profile matches a configurable strategy.

### 3. Sandwich simulator

A self‑contained Rust module:

- Implements x·y = k simulation with fees.
- Provides:
  - `simulate_swap_xyk` for a single swap.
  - `simulate_sandwich_sol_in` for front + opportunity + back sequence.
- Returns a `SandwichResult` struct with:
  - `profit_sol`
  - `front_token_out`
  - `opportunity_effective_price`
  - `opportunity_price_impact`
  - final reserves (for debugging).

This can be unit‑tested in isolation and reused for other AMMs.

### 4. Jito integration

A small configuration layer:

- `JitoMode` enum: `Off`, `DryRun`, `Live`.
- `JitoConfig` struct:
  - Max tip per bundle.
  - Aggregate tip cap per process.
  - Aggregate **front notional** cap per process.
  - Minimum net profit in SOL for a bundle to be considered.
  - Block engine endpoint URL.

When `JitoMode::DryRun`:

- The bot still does full planning and simulation, but **does not** submit
  bundles; it logs what it *would* have done for calibration.

When `JitoMode::Live`:

- It actually serializes and submits bundles to the Jito block engine.

---

## Configuration (sanitized)

All sensitive details are **environment‑driven**. Typical env vars:

```bash
# Core RPC & price feeds
export RPC_URL="https://your-rpc-or-local-validator"
export PYTH_HERMES_URL="https://hermes.pyth.network"
export PYTH_SOL_USD_ID="0x..."

# Raydium pool source
export RAYDIUM_LIQUIDITY_URL="https://api.raydium.io/v2/sdk/liquidity/mainnet.json"

# Jito
export JITO_BLOCK_ENGINE_URL="https://<region>.block-engine.jito.wtf/api/v1/bundles"
export JITO_MODE="DryRun"    # Off | DryRun | Live

# MEV wallet (never committed to git)
export MEV_KEYPAIR_PATH="/path/to/mev-keypair.json"

# MEV token accounts (examples; use your own ATAs)
export MEV_WSOL_ATA="<WSOL ATA for your MEV wallet>"
export MEV_USDC_ATA="<USDC ATA for your MEV wallet>"

# Risk & capital allocation (tuned offline)
export JITO_MAX_TIP_SOL="..."
export JITO_SESSION_TIP_CAP_SOL="..."
export JITO_SESSION_NOTIONAL_CAP_SOL="..."
