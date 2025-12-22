# Solana Price-Based Trading Stack (Yellowstone gRPC + Metis + Local Node)

> ⚠️ DO NOT COMMIT: private keys, API keys, auth headers, IPs/hostnames, or wallet addresses.
> Keep all secrets in shell exports / dotenv that is **gitignored**.

This repo is a **price/quote–based trading system** for Solana:
- **No sandwiching**
- **No backrunning / victim-tx dependence**
- **No mempool/shredstream requirements**
- Only **self-initiated trades** driven by periodic scanning + risk controls

The system is designed to run on an Ubuntu bare-metal server with:
- A **local Solana RPC node** (Agave / solana-validator build)
- **Yellowstone gRPC (Geyser plugin)** for low-latency account/tx/slot streaming
- **Jupiter Metis (self-hosted)** as the routing/quote engine (avoids hosted Jupiter rate limits)
- Optional: **Jito bundles** for atomic multi-tx execution + inclusion priority
- Optional: **Bloxroute Enterprise** for additional network connectivity and/or market-signal feeds

---

## 1) What “Metis” is (and isn’t)

Metis is a routing engine used to compute best paths across Solana DEX liquidity and build swap transactions based on quotes. Self-hosting Metis requires a **Binary Key**, which (as of the current Jupiter guidance) is gated by **10,000 staked JUP** per instance. :contentReference[oaicite:0]{index=0}

Important operational point:
- **Metis produces quotes and swap transaction payloads**
- You still **sign locally** with your wallet and **send** via RPC (or bundle via Jito) :contentReference[oaicite:1]{index=1}

That means your trading key never needs to live inside Metis itself.

---

## 2) Why Yellowstone gRPC exists in this stack

Yellowstone gRPC is a **Solana Geyser plugin** that streams **accounts/transactions/blocks/slots** with lower latency and lower “RPC polling load” than constantly re-fetching everything by HTTP RPC. :contentReference[oaicite:2]{index=2}

In a scanning-based trader, it’s used to:
- Keep local state fresh (vault balances, pool state, token account changes)
- Reduce heavy periodic `getMultipleAccounts` loops
- Trigger “re-scan now” when a watched account changes (optional)

---

## 3) High-level architecture
         ┌──────────────────────────┐
         │   Agave local node RPC   │
         │   (simulate + send tx)   │
         └───────────┬──────────────┘
                     │
             (Geyser plugin)
                     │
         ┌───────────▼──────────────┐
         │     Yellowstone gRPC      │
         │  accounts / tx / slots    │
         └───────────┬──────────────┘
                     │
         ┌───────────▼──────────────┐
         │     Metis (self-hosted)   │
         │   quotes + route building │
         └───────────┬──────────────┘
                     │
         ┌───────────▼──────────────┐
         │      Rust trader bot      │
         │ scan → risk-gate → sim →  │
         │ sign → send / bundle      │
         └───────────┬──────────────┘
                     │
    ┌────────────────▼───────────────┐
    │ Optional: Jito block engine     │
    │ bundles for atomic multi-tx     │
    └────────────────────────────────┘

Optional sidecar:

Bloxroute gateway / WS feeds (signals / connectivity)


---

## 4) Repo layout (current)

Typical layout in this crate:

- `src/bin/arb.rs`
  - Main trading binary (name historical; functionally this is the price/quote trader)
- `src/bin/fund_usdc.rs`
  - Utility to fund USDC using Jupiter swap API (or later: Metis local)
- `src/bin/fund_jup.rs`
  - Utility to buy JUP (for staking / tooling workflows)

You can add additional scanning binaries later as separate `src/bin/*.rs` programs without touching the core runtime.

---

**DELIBERATELY SKIPPING 5 and 6.  Too sensitive for Public forums.
**

7) Operating model: scanning-based price/quote trading

Instead of reacting to “pending” transactions, this stack finds opportunities by repeatedly answering:

What is the best executable route right now? (Metis quote)

What’s the expected output after slippage + fees + tips?

Does a risk-bounded trade clear the threshold?

Simulate

Sign

Send (RPC or Jito bundle)

Metis provides both the quote and the ability to build the swap transaction payload; you sign & send. 
Jupiter Developers
+1

Strategy modules (planned)

Run them as separate binaries or subcommands:

Cross-venue price arb (spatial): same pair, different venues

Triangular cycles: A→B→C→A using a token graph (petgraph)

Oracle-relative: pool price deviates from oracle (only where you can hedge/exit safely)

LST relative value: correlated assets, z-score based

8) Logging / observability

Recommended log tags (examples):

[SCAN] scanning iteration start/end, how many pairs evaluated

[QUOTE] quote received (route labels, impact, in/out)

[SKIP] why a candidate was rejected (impact too high, profit too low, stale state, etc.)

[SIM] simulation summary / failure reason

[SEND] signed tx signature or bundle id

[ACCT] accounting line: gross, fees, tip, net, ROI

Example grep:

tail -F arb.log | grep --line-buffered -E \
'SCAN|QUOTE|CANDIDATE|SKIP|SIM|SEND|ACCT|ERROR|WARN|JITO|BLOX'
