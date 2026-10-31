# AutomatedTrader

A real-time intraday position management system for Fyers broker — automated SL, Target, and Trailing Stop-Loss execution built with **Rust** (backend) and **React** (frontend).

## What It Does

- Monitors your open Fyers intraday positions in real-time
- Auto-applies Stop-Loss and Target rules to every new trade
- Trails the SL upward for every 0.5% profit gain (locking in profits)
- Executes market orders automatically when SL or Target is hit
- Shows live Nifty/Sensex prices + PCR on the dashboard
- Manual controls: Square Off, Trail SL to Zero, Jump Target, EXIT ALL

## SpecKit Workflow

This project follows **spec-driven development** using [SpecKit](https://github.com/github/spec-kit).

### Documents

| Document | Purpose |
|----------|---------|
| [constitution.md](./constitution.md) | Architectural rules, safety principles, tech stack — governs everything |
| [Spec 1 — Auth](./specifications/01-auth/spec.md) | OAuth2 login with Fyers, session management |
| [Spec 2 — Position Monitoring](./specifications/02-position-monitoring/spec.md) | Real-time position tracking, new trade detection |
| [Spec 3 — Price Feed](./specifications/03-price-feed/spec.md) | Live price feed, P/L engine, PCR calculation |
| [Spec 4 — SL/Target Engine](./specifications/04-sl-target-engine/spec.md) | Trailing SL, automated execution, manual overrides |
| [Spec 5 — React Dashboard](./specifications/05-react-dashboard/spec.md) | Full frontend: Indices, Trades Grid, action buttons |

### SpecKit Steps (per specification)

```
1. spec.md     ✅ Written (this step — all 5 done)
2. plan.md     → /speckit-plan  (technical architecture per spec)
3. tasks.md    → /speckit-tasks (granular implementation checklist)
4. implement   → /speckit-implement (write code from tasks)
5. converge    → /speckit-converge (validate against spec + tests)
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend language | Rust (stable) |
| Async runtime | Tokio |
| HTTP server | Actix-Web |
| WebSocket client | tokio-tungstenite |
| Frontend | React 18 + TypeScript (Vite) |
| Data grid | AG Grid Community |
| State management | Zustand |
| Broker | Fyers API v3 |

## Prerequisites

- Fyers API app credentials (`app_id`, `secret_key`) from [myapi.fyers.in](https://myapi.fyers.in)
- Rust (stable toolchain via `rustup`)
- Node.js 18+
- A Fyers trading account with API access enabled