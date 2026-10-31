# Feature Specification: Real-time Price Feed & P/L Engine

**Feature Branch**: `03-price-feed-pl`

**Created**: 2026-10-31 | **Revised**: 2026-10-31 (v2 — gap resolutions applied)

**Status**: Draft

---

## Gap Resolutions Applied
- **GAP 1 resolved**: PCR is **not implemented in v1**. Indices section shows live price and change % only (no PCR field).
- **GAP 2 resolved**: P/L % is computed on notional: `pl_pct = pl / (avgEntryPrice × abs(qty)) × 100`.
- **GAP 4 resolved**: Binary WebSocket frames are parsed by porting the Fyers `fyers-apiv3` Python SDK binary frame spec to Rust using `byteorder` + `bytes`.
- **GAP 7 resolved**: Price feed is suspended outside market hours; last known values are retained for display as "stale".

---

## User Scenarios & Testing

### User Story 1 — Binary Frame Parsing from Fyers Data Socket (Priority: P1)

The system correctly parses every binary market data frame arriving from the Fyers Data WebSocket into structured tick data, with zero data corruption.

**Why this priority**: Incorrect price = incorrect SL/Target firing = real financial loss. This is the most critical low-level correctness requirement.

**Independent Test**: Feed 500 recorded binary frames (captured from Fyers Python SDK test runs) through the Rust binary frame parser; compare output LTP values against the Python SDK's parsed output with 0 mismatches.

**Acceptance Scenarios**:

1. **Given** a 36-byte binary frame for `NSE:NIFTY24NOV22000CE` arrives on the Data Socket, **When** parsed, **Then** the resulting `Tick` struct contains correct `symbol`, `ltp`, `open`, `high`, `low`, `close`, `volume`, and `timestamp`.
2. **Given** a malformed/short binary frame (< expected bytes), **When** parsing is attempted, **Then** the error is logged with the raw bytes, the frame is discarded, a `parse_error` counter is incremented, and the main tick loop continues without panicking.
3. **Given** 1000 binary frames arrive per second for 10 symbols, **When** processed, **Then** zero frames are dropped and the parser throughput does not become a bottleneck (verified by benchmark: ≥ 50,000 frames/sec single-threaded parse speed).

---

### User Story 2 — Live P/L per Position in Real-time (Priority: P1)

Every position in the dashboard shows its running P/L in ₹ and %, updating tick-by-tick.

**Independent Test**: Subscribe a mock BUY position (entry=1000, qty=100) to the price feed, inject tick at LTP=1010, verify `PLUpdate { pl=1000.0, pl_pct=1.0 }` is published to the event bus and broadcast to frontend.

**Acceptance Scenarios**:

1. **Given** BUY position (entry=1000, qty=100), **When** tick at LTP=1010 arrives, **Then** `pl = (1010-1000) × 100 = ₹1,000`, `pl_pct = 1000 / (1000 × 100) × 100 = 1.00%`.
2. **Given** SELL position (entry=500, qty=75), **When** tick at LTP=490, **Then** `pl = (500-490) × 75 = ₹750` (short profits on decline), `pl_pct = 750 / (500 × 75) × 100 = 2.00%`.
3. **Given** a tick arrives for symbol `NSE:RELIANCE-EQ`, **When** only one position holds that symbol, **Then** only that position's P/L is recomputed — no other positions are touched.
4. **Given** the registry has a new position subscribed, **When** the next Data Socket frame for that symbol arrives, **Then** P/L is computed and broadcast (subscription → first tick ≤ 500ms).

---

### User Story 3 — Nifty 50 and Sensex Live Index Feed (Priority: P1)

The indices bar at the top shows live Nifty and Sensex prices with change amount and change % — updating in real-time. No PCR displayed in v1.

**Independent Test**: Inject a mock index tick for `NSE:NIFTY50-INDEX`, verify frontend receives `INDEX_UPDATE` with correct `ltp`, `change`, `change_pct`.

**Acceptance Scenarios**:

1. **Given** the app is connected, **When** a tick for `NSE:NIFTY50-INDEX` arrives, **Then** frontend receives `{ type: "INDEX_UPDATE", symbol: "NSE:NIFTY50-INDEX", displayName: "NIFTY 50", ltp: 24250.75, change: 125.30, change_pct: 0.52 }`.
2. **Given** both index symbols are subscribed at startup, **When** Data WebSocket reconnects, **Then** index symbols are re-subscribed within 1 second alongside position symbols.
3. **Given** the market is closed (after 15:30 IST), **When** displayed, **Then** index values show last known price with a "Market Closed" indicator instead of live updates.

---

### User Story 4 — WebSocket Reconnect with Subscription Recovery (Priority: P1)

The Fyers Data WebSocket drops. The system auto-reconnects and re-subscribes all symbols without any data loss in the position monitoring logic.

**Acceptance Scenarios**:

1. **Given** the Data WebSocket disconnects, **When** detected, **Then** reconnect attempts begin with exponential backoff (1s, 2s, 4s… max 30s).
2. **Given** reconnect succeeds, **When** connected, **Then** all current position symbols AND index symbols are re-subscribed within 1 second.
3. **Given** reconnect is in progress, **When** the frontend is shown, **Then** a "Reconnecting…" indicator is shown for the data feed; last known P/L values remain visible (not zeroed out).

---

### Edge Cases

- Tick arrives for symbol not in position registry → discard silently, no processing.
- LTP = 0 or LTP > 10x the previous LTP (erroneous spike) → log warning with `[SUSPICIOUS_TICK]` prefix, skip P/L update and SL/Target evaluation for that tick.
- Data Socket delivers same symbol tick twice in same millisecond → deduplicate by timestamp; only process the later one.
- Rate-limited frontend broadcast: if 10 ticks arrive for the same symbol within 100ms, only the last tick's values are broadcast to the frontend (debounce, not drop — SL/Target engine sees all ticks).
- `change` and `change_pct` for indices: computed as `(ltp - prev_close)` where `prev_close` comes from the first tick's `close` field on subscription. If unavailable, show `change: 0, change_pct: 0`.

---

## Requirements

### Functional Requirements

- **FR-001**: System MUST establish a Fyers Data WebSocket connection on login/startup using `tokio-tungstenite`.
- **FR-002**: System MUST subscribe to `NSE:NIFTY50-INDEX` and `BSE:SENSEX-INDEX` at startup unconditionally.
- **FR-003**: System MUST subscribe to every symbol in `PositionRegistry`; subscriptions MUST update dynamically as positions are added or removed.
- **FR-004**: System MUST implement a binary frame parser ported from the Fyers Python SDK (`fyers-apiv3`) binary format. Parser MUST produce: `Tick { symbol, ltp, open, high, low, close, volume, timestamp }`.
- **FR-005**: Parser MUST be a pure function with no I/O — takes `&[u8]`, returns `Result<Tick, ParseError>`. Unit-testable with raw byte fixtures.
- **FR-006**: On each tick, system MUST compute: `pl = (ltp - avg_entry_price) × qty × direction` (direction=+1 for BUY, -1 for SELL). `pl_pct = pl / invested_amount × 100`.
- **FR-007**: P/L updates MUST be published to the internal `EventBus` for consumption by the SL/Target engine (Spec 4) **synchronously** on the hot path (no channel overhead for this critical path).
- **FR-008**: Frontend WebSocket P/L broadcasts MUST be debounced: at most 1 update per symbol per 100ms sent to frontend. The SL/Target engine sees every tick.
- **FR-009**: Index updates MUST be broadcast to frontend separately from position P/L updates using event type `INDEX_UPDATE`.
- **FR-010**: Data WebSocket MUST auto-reconnect with exponential backoff (1s initial, 2× multiplier, 30s max, ±20% jitter). On reconnect, re-subscribe all active symbols and index symbols within 1 second.
- **FR-011**: Outside market hours (before 09:15 / after 15:30 IST), system MUST NOT attempt Data WebSocket reconnects; existing connection is closed gracefully at 15:31 IST.
- **FR-012**: System MUST expose `GET /api/market/snapshot` returning the current `{ positions: [{ position_id, symbol, ltp, pl, pl_pct }], indices: [{ symbol, ltp, change, change_pct }] }` for frontend reconnect scenarios.
- **FR-013**: A `parse_error` counter MUST be tracked and exposed via `GET /api/health` for observability.
- **FR-014**: PCR is **not implemented in v1**. No option chain calls. No PCR field in any response.

### Key Entities

- **Tick**: `{ symbol: String, ltp: f64, open: f64, high: f64, low: f64, close: f64, volume: u64, timestamp: u64 }` — output of binary parser.
- **PLUpdate**: `{ position_id: String, symbol: String, ltp: f64, pl: f64, pl_pct: f64, timestamp: u64 }` — published to EventBus.
- **IndexUpdate**: `{ symbol: String, display_name: String, ltp: f64, change: f64, change_pct: f64 }` — broadcast to frontend.
- **DataSocketState**: `CONNECTING | CONNECTED | RECONNECTING | CLOSED`.
- **Subscription**: `HashSet<String>` of currently subscribed symbols (positions + indices).

---

## Success Criteria

- **SC-001**: Tick → P/L computation → EventBus publish latency ≤ 5ms (hot path, measured in benchmarks).
- **SC-002**: Tick → frontend WebSocket broadcast latency ≤ 100ms (debounce window + network).
- **SC-003**: Binary parser handles 50,000 frames/second without being the bottleneck (Criterion benchmark).
- **SC-004**: Binary parser output matches Fyers Python SDK output for 500 recorded frames (0 mismatches in regression test).
- **SC-005**: Data WebSocket reconnect + re-subscription completes within 2 seconds of disconnect detection.
- **SC-006**: `GET /api/market/snapshot` returns current state within 50ms.

---

## Assumptions

- **Binary frame format**: Will be ported from `fyers-apiv3` Python SDK source code at `fyers_api/data_socket.py`. The format uses fixed-length struct packing (big-endian). Exact byte layout confirmed at implementation time.
- Data Socket URL: `wss://api.fyers.in/socket/v2` — confirmed at implementation time from Python SDK.
- Subscription message format: `{"T": "SUB_L2", "L2LIST": ["NSE:NIFTY50-INDEX", ...], "SUB_T": 1}` — confirmed from Python SDK.
- For indices, `change` = `ltp - prev_close` where `prev_close` is captured from the first tick's `close` field on each trading day.
- Sensex (`BSE:SENSEX-INDEX`) availability on Fyers Data Socket to be confirmed at implementation; if unavailable, show "N/A" for Sensex.
- v1 does **not** implement TBT (Tick-by-Tick) socket — Data Socket is sufficient. TBT can be added in v2 if Data Socket latency proves insufficient.
