# Feature Specification: Position Monitoring & Trade Detection

**Feature Branch**: `02-position-monitoring`

**Created**: 2026-10-31 | **Revised**: 2026-10-31 (v2 — gap resolutions applied)

**Status**: Draft

---

## Gap Resolutions Applied
- **GAP 2 resolved**: "Amount Invested" = `avgEntryPrice × abs(quantity)` (notional value) — consistent for both equity and F&O.
- **GAP 7 resolved**: Outside market hours (before 09:15 / after 15:30 IST), position monitors are paused; order execution is suspended; last known state is shown as "stale".
- **SQLite scope**: Position state checkpointed every 5 seconds and on every state change. On backend restart, positions are recovered from checkpoints.

---

## User Scenarios & Testing

### User Story 1 — Position Recovery on Backend Restart (Priority: P1)

The backend restarts during the trading day (e.g., crash or deliberate restart). Within seconds of restart, all active positions are restored from SQLite checkpoints with their current SL prices, target prices, and trailing high-water marks — as if nothing happened.

**Why this priority**: Without checkpoint recovery, a restart during market hours means all SL/Target rules are lost — catastrophic for a live trading system.

**Independent Test**: Write 3 position checkpoint rows to SQLite (with custom SL/target values), start the backend, call `GET /api/positions` — verify all 3 positions are in the registry with the checkpointed SL/target values, not recalculated from scratch.

**Acceptance Scenarios**:

1. **Given** SQLite `position_checkpoints` has 2 rows with `status=OPEN`, **When** the backend starts, **Then** both positions are loaded into `PositionRegistry` within 1 second of startup.
2. **Given** a checkpoint has `sl_price=980, target_price=1030, trailing_high_water=0.5`, **When** loaded into the registry, **Then** the position's `PositionMonitor` task starts with exactly these values — it does NOT recalculate SL/Target from scratch.
3. **Given** a checkpoint has `status=SQUAREDOFF`, **When** loaded, **Then** it is NOT added to the registry (already closed — skip it).
4. **Given** Fyers positions API shows a position that is NOT in SQLite checkpoints (new trade placed while backend was down), **When** reconciliation runs post-startup, **Then** it is added as a new position and default SL/Target rules are applied.

---

### User Story 2 — Initial Position Snapshot on Login (Priority: P1)

After logging in (or after silent startup auth), the system fetches all current open intraday positions from Fyers and merges them with SQLite checkpoints to build the authoritative position registry.

**Independent Test**: Mock Fyers `/positions` returning 2 positions; SQLite has checkpoints for 1 of them. Verify: the checkpointed position uses stored SL/target values; the new position gets default SL/target applied.

**Acceptance Scenarios**:

1. **Given** the user logs in with 3 open Fyers positions, **When** position sync runs, **Then** positions are loaded, `investedAmount = avgEntryPrice × abs(qty)` computed for each, and full list pushed to frontend within 2 seconds.
2. **Given** a Fyers position matches a SQLite checkpoint, **When** merged, **Then** the checkpoint's `sl_price`, `target_price`, `initial_target_price`, and `trailing_high_water` are used — not recalculated.
3. **Given** a Fyers position has no SQLite checkpoint, **When** added, **Then** default SL/Target from `user_config` is applied (from SQLite `user_config` table, with hardcoded fallback SL=2%, Target=3%).

---

### User Story 3 — Real-time Detection of New Trades (Priority: P1)

The trader takes a new trade on the Fyers terminal. The new position appears in the dashboard within 1 second and automated SL/Target rules are applied immediately.

**Acceptance Scenarios**:

1. **Given** the General WebSocket delivers an order event with `status=FULLY_TRADED`, **When** processed, **Then** the new position is added to the registry, a SQLite checkpoint is written, a `PositionMonitor` task is spawned, and `POSITION_ADDED` is broadcast to the frontend.
2. **Given** the new position is added, **When** `investedAmount` is computed, **Then** `investedAmount = avgEntryPrice × abs(qty)`.
3. **Given** an existing position has additional quantity added (pyramid), **When** the fill arrives, **Then** `avgEntryPrice` and `qty` are updated in the registry AND the SQLite checkpoint, and SL/Target are recalculated from the new average entry.

---

### User Story 4 — Periodic Reconciliation & Checkpoint Sync (Priority: P2)

Every 30 seconds, the system reconciles the in-memory registry against Fyers positions API AND writes all dirty position states to SQLite as checkpoints.

**Acceptance Scenarios**:

1. **Given** a position is closed on the Fyers terminal (not via this app), **When** the 30-second reconciliation runs, **Then** it is removed from the registry, its SQLite checkpoint is marked `status=SQUAREDOFF`, its `PositionMonitor` is cancelled, and `POSITION_REMOVED` is broadcast.
2. **Given** 5 positions are active with changing SL values (from trailing), **When** the 5-second checkpoint sync fires, **Then** all 5 SQLite `position_checkpoints` rows are updated with the latest values.
3. **Given** a position has `status=SQUARING_OFF` (order in flight), **When** reconciliation sees it closed on Fyers, **Then** it confirms the close, marks checkpoint `SQUAREDOFF`, and notifies frontend.

---

### User Story 5 — Market Close Behaviour (Priority: P2)

At 15:30 IST, the system detects market close, suspends all automated order execution, and marks all open position data as "stale" for display purposes.

**Acceptance Scenarios**:

1. **Given** the system clock reaches 15:30 IST, **When** market close is detected, **Then** all `PositionMonitor` tasks receive a `MARKET_CLOSED` signal and suspend SL/Target execution logic.
2. **Given** market is closed, **When** the frontend is shown, **Then** each position row has a "Stale" indicator and all action buttons (Square Off, Trail SL to Zero, Jump Target, EXIT ALL) are disabled.
3. **Given** market closes with open positions, **When** SQLite checkpoints are written at close, **Then** they are available for recovery if the system restarts before the next market open.

---

### Edge Cases

- General WebSocket disconnects → monitors continue with last known state; on reconnect, immediate reconciliation runs.
- Race condition: two fill events for same position arrive simultaneously → `DashMap` atomic update + per-position mpsc channel ensures sequential processing.
- `avgEntryPrice` is zero or missing from Fyers API → log warning, skip automated SL/Target for that position, flag it as `NEEDS_MANUAL_REVIEW` in the registry and frontend.
- SQLite write fails during checkpoint → log error, retry once after 1 second; if still failing, continue in-memory (log critical alert to frontend).
- Position `productType` is not `INTRADAY` (e.g., `CNC` delivery trade) → include in registry for display but flag as "Non-intraday" and disable automated SL/Target (manual intervention only).

---

## Requirements

### Functional Requirements

- **FR-001**: On startup (after auth), system MUST load all `position_checkpoints` with `status=OPEN` from SQLite into `PositionRegistry`.
- **FR-002**: System MUST call `GET /api/v3/positions` and merge results with SQLite checkpoints: checkpointed positions retain stored SL/target/trailing state; new positions get default rules applied.
- **FR-003**: `investedAmount` MUST be computed as `avgEntryPrice × abs(quantity)` for all position types.
- **FR-004**: System MUST connect to Fyers General WebSocket for real-time order status events.
- **FR-005**: On receiving a General WebSocket order event with `status=FULLY_TRADED`, system MUST add/update the position in the registry and write a SQLite checkpoint within 500ms.
- **FR-006**: For each new position, system MUST spawn a dedicated `PositionMonitor` Tokio task; for closed positions, the task MUST be cancelled and checkpoint updated to `SQUAREDOFF`.
- **FR-007**: System MUST run a reconciliation cycle every 30 seconds (Fyers positions API vs. registry).
- **FR-008**: System MUST run a checkpoint sync every 5 seconds (registry state → SQLite for all dirty positions).
- **FR-009**: System MUST broadcast position events to frontend WebSocket clients: `POSITIONS_SNAPSHOT`, `POSITION_ADDED`, `POSITION_UPDATED`, `POSITION_REMOVED`.
- **FR-010**: At 15:30 IST, system MUST broadcast `MARKET_CLOSED` to frontend and suspend automated order execution in all `PositionMonitor` tasks.
- **FR-011**: At 09:15 IST, system MUST broadcast `MARKET_OPEN` to frontend and resume automated order execution.

### Key Entities

- **PositionState** (in-memory + SQLite): `{ position_id, symbol, exchange, qty: i32, side: Side, avg_entry_price: f64, invested_amount: f64, product_type: String, sl_price: Option<f64>, target_price: Option<f64>, initial_target_price: Option<f64>, trailing_high_water: f64, status: PositionStatus, updated_at: DateTime<Utc> }`.
- **PositionStatus**: `OPEN | SQUARING_OFF | SQUAREDOFF | NEEDS_MANUAL_REVIEW`.
- **PositionRegistry**: `Arc<DashMap<String, PositionState>>`.
- **SQLite `position_checkpoints`**: Mirrors `PositionState`, written every 5 seconds and on every state transition.

---

## Success Criteria

- **SC-001**: Position recovery from SQLite on restart completes within 1 second of startup.
- **SC-002**: New trade detected and in dashboard within 1 second of General WebSocket fill event.
- **SC-003**: Checkpoint sync writes all dirty positions to SQLite within 5 seconds of state change.
- **SC-004**: Reconciliation correctly handles 100% of discrepancy cases in integration tests.
- **SC-005**: All position monitors correctly suspend at 15:30 IST (verified by time-mocked unit tests).

---

## Assumptions

- Fyers positions API returns `productType` field distinguishing `INTRADAY` from `CNC`; v1 supports both for display but only auto-manages `INTRADAY`.
- `avgEntryPrice` is provided by Fyers positions API and is non-zero for valid open positions.
- General WebSocket order events include `status`, `symbol`, `qty`, `avgPrice`, `side` fields.
- SQLite `position_checkpoints` table uses `position_id` as primary key (upsert on conflict).
- The "5-second checkpoint sync" is implemented as a separate Tokio task (not on the tick hot path).
- `trailing_high_water` default = 0.0 (no favorable move yet) for new positions.
