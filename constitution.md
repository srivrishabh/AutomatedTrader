# AutomatedTrader Constitution

## Core Principles

### I. Safety-First (NON-NEGOTIABLE)
This system places real money orders on a live broker. Every design decision MUST prioritize safety:
- All trading operations MUST be idempotent — duplicate order placement is strictly forbidden
- Every order placed MUST be immediately persisted to SQLite with: timestamp (nanosecond), symbol, side, quantity, order type, trigger reason, broker response, and status
- Stop-loss and target execution MUST use **market orders only** — no limit orders for automated exits
- System MUST have a global "kill switch" that halts all automated order placement instantly
- No order may be placed without a valid, unexpired `access_token` — all auth checks are pre-flight
- Before any exit order, system MUST verify the position is still open via the in-memory registry (no double-exits)
- The SL/Target status for each position MUST be persisted as a checkpoint so a restart cannot result in a missed or duplicate trigger

### II. Layered Architecture (NON-NEGOTIABLE)
The backend MUST follow strict layering with zero cross-layer violations:

```
API Layer        → HTTP handlers + WebSocket server (actix-web, actix-ws)
                   Responsibility: Request parsing, auth middleware, response serialization

Service Layer    → Business logic orchestration
                   Responsibility: Coordinates domain objects, calls infrastructure, drives persistence

Domain Layer     → Core trading rules (P/L calc, SL/Target/Trailing logic, position state)
                   Responsibility: Pure functions, no I/O, no external dependencies, fully unit-testable

Infrastructure   → Fyers REST client (thin), Fyers WebSocket client (binary parser), SQLite persistence
                   Responsibility: All external I/O, hidden behind traits
```

**Trait Abstraction Rule**: Every infrastructure component MUST be behind a Rust trait (`BrokerClient`, `MarketDataFeed`, `PersistenceStore`). Tests inject mocks; production uses real implementations.

### III. Concurrency & Fault-Tolerant Architecture
- Rust async runtime: **Tokio** exclusively — no blocking calls on async threads
- Each open position runs in its **own dedicated Tokio task** (`PositionMonitor`), communicating via `mpsc` channels
- Shared hot state: `Arc<DashMap<PositionId, PositionState>>` — prefer DashMap for lock-free concurrent reads
- Message passing is preferred over shared memory — `PositionMonitor` tasks send events to a central `EventBus`
- **No `unwrap()` or `expect()` in production paths** — all errors propagate via `Result<_, AppError>`
- Periodic **checkpoint writes** to SQLite every 5 seconds for all active position states (SL price, target price, trailing high-water mark, status)
- On startup, **recover position states** from SQLite checkpoints to survive backend restarts
- Reconnection strategy: exponential backoff for all WebSocket connections (initial: 1s, max: 30s, jitter: ±20%)

### IV. Persistence Layer (SQLite — High-Water Mark & Checkpoint)
SQLite is the system's fault-tolerance backbone. It serves as:
1. **Token Store**: `access_token`, `client_id`, `expiry_timestamp` — loaded on startup, eliminates daily re-auth if token still valid
2. **User Config Store**: `default_sl_pct`, `default_target_pct`, `trailing_step_pct` — persisted across restarts
3. **Position State Checkpoint**: Every active position's SL price, target price, trailing high-water mark, initial target price, position status — written every 5 seconds and on every state change
4. **Audit Log**: Every order placed (automated or manual) with full context — immutable append-only, never deleted

Schema summary:
```sql
tokens(id, access_token, client_id, expiry_timestamp, created_at)
user_config(key TEXT PRIMARY KEY, value TEXT)
position_checkpoints(position_id, symbol, side, qty, avg_entry_price, sl_price, target_price,
                     initial_target_price, trailing_high_water, status, updated_at)
order_audit(id, position_id, symbol, side, qty, order_type, trigger_type, trigger_price,
            ltp_at_trigger, broker_order_id, broker_response, status, placed_at)
```

### V. Test-First Quality Gate
- Domain Layer (P/L calculation, trailing SL math, SL/Target hit detection) MUST have unit test coverage ≥ 90%
- All broker interactions hidden behind traits → integration tests use mock broker, never place real orders
- SQLite operations tested with in-memory SQLite (`rusqlite` in-memory mode)
- `cargo clippy -- -D warnings` must pass with zero warnings before any implementation is merged
- `cargo fmt --check` enforced — all code formatted via `rustfmt`

### VI. Low-Latency Principles
- Price ticks from Fyers WebSocket → SL/Target decision → Order placement: target latency < 100ms end-to-end
- Tick processing is purely in-memory on the hot path — SQLite writes are async and off the hot path
- Binary WebSocket frames parsed using the ported Fyers Python SDK binary frame spec (struct-level porting)
- Order placement is async and non-blocking — the tick processing loop MUST NOT wait for order confirmation
- Frontend WebSocket updates broadcast at maximum every 100ms per symbol (rate-limited via debounce)

### VII. Market Hours Awareness
- System is considered **active** between 09:15 and 15:30 IST on trading days
- Outside market hours: order buttons disabled, last known prices shown as "stale", "Market Closed" banner visible
- Automated SL/Target execution is **suspended** outside market hours (no market orders when market is closed)
- WebSocket connections to Fyers are maintained but reconnect attempts are paused outside market hours
- Market open/close events broadcast to frontend via WebSocket

## Technology Stack

### Backend (Rust)
| Concern | Crate | Notes |
|---------|-------|-------|
| Async runtime | `tokio` | Full features |
| HTTP server | `actix-web` | Latest stable |
| Backend WS server | `actix-ws` | Per-client WebSocket |
| HTTP client (Fyers REST) | `reqwest` (rustls) | TLS, async |
| WebSocket client (Fyers) | `tokio-tungstenite` | Binary frame support |
| JSON serialization | `serde` + `serde_json` | Derive macros |
| Concurrent hashmap | `dashmap` | Lock-free reads |
| Structured logging | `tracing` + `tracing-subscriber` | JSON output for prod |
| Error handling | `thiserror` + `anyhow` | Typed + contextual errors |
| Config management | `config` crate | TOML + env override |
| Binary frame parsing | `bytes` + `byteorder` | Zero-copy where possible |
| SQLite persistence | `rusqlite` + `tokio` | Async wrapper via `tokio::task::spawn_blocking` |
| Time handling | `chrono` | IST timezone, market hours check |

### Frontend (React)
| Concern | Library | Notes |
|---------|---------|-------|
| Framework | React 18 + TypeScript | Vite build tool |
| State management | Zustand | Lightweight, no Redux boilerplate |
| Data grid | AG Grid Community | High-performance, real-time row updates |
| WebSocket | Native browser WebSocket | Direct to backend `/ws` |
| HTTP client | Axios | For action button REST calls |
| Routing | React Router v6 | Protected routes |

### Workspace Structure
```
AutomatedTrader/
├── constitution.md
├── specifications/
│   ├── 01-auth/spec.md
│   ├── 02-position-monitoring/spec.md
│   ├── 03-price-feed/spec.md
│   ├── 04-sl-target-engine/spec.md
│   └── 05-react-dashboard/spec.md
├── backend/                    # Rust workspace
│   ├── Cargo.toml
│   ├── crates/
│   │   ├── fyers-client/       # Thin Fyers REST + WS client (infra)
│   │   ├── trading-engine/     # Domain: pure P/L, SL, trailing logic
│   │   ├── persistence/        # SQLite store: tokens, config, checkpoints, audit
│   │   ├── position-monitor/   # Service: registry + per-position Tokio tasks
│   │   └── server/             # API layer: actix-web + WebSocket server
│   └── tests/
└── frontend/                   # React Vite app
```

## Fyers API Constraints
- **Base URL**: `https://api-t1.fyers.in/api/v3/`
- **Auth header format**: `Authorization: {app_id}:{access_token}`
- **Thin client scope**: Only the following endpoints are wrapped — auth (validate-authcode), positions, order placement (sync), order book, General WebSocket, Data WebSocket
- **Rate limits**: Max 5 order placements/second; EXIT ALL uses a semaphore of 3 concurrent order requests
- **WebSocket binary protocol**: Ported from `fyers-apiv3` Python SDK binary frame parser
- **PCR**: Not implemented in v1 — indices show only live price and change %
- **Market hours**: 09:15–15:30 IST; system enforces this at the service layer

## Resolved Gap Decisions (ADRs)

| Gap | Decision |
|-----|----------|
| PCR data | Skipped in v1 — indices show price + change % only |
| Amount Invested | Notional = `avgEntryPrice × abs(qty)` — simple, consistent |
| Rust SDK | Thin client — only wrap endpoints actually needed |
| Binary WS protocol | Port from Fyers Python SDK binary frame spec |
| Token expiry | Re-login modal in-app + SQLite persistence (load on startup if valid) |
| Rate limits / EXIT ALL | Semaphore of 3 concurrent order requests |
| Market hours | Stale badge + Market Closed banner + disable order buttons |
| SQLite scope | Token + user config + position checkpoints + order audit log |

## Governance
- This Constitution supersedes all other design preferences
- Any deviation requires explicit documentation in the spec with a rationale
- All PRs/implementations must be verified against this Constitution before proceeding

**Version**: 2.0.0 | **Ratified**: 2026-10-31 | **Last Amended**: 2026-10-31
