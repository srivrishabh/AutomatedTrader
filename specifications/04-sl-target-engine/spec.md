# Feature Specification: SL / Target / Trailing Stop-Loss Engine

**Feature Branch**: `04-sl-target-engine`

**Created**: 2026-10-31 | **Revised**: 2026-10-31 (v2 — gap resolutions applied)

**Status**: Draft

---

## Gap Resolutions Applied
- **GAP 2 resolved**: SL/Target % applied on notional `investedAmount = avgEntryPrice × abs(qty)`.
- **GAP 3 resolved**: Order placement uses thin Fyers REST client (reqwest) wrapping `POST /api/v3/orders/sync`.
- **GAP 6 resolved**: EXIT ALL uses `tokio::sync::Semaphore` with 3 permits — max 3 concurrent order requests at a time.
- **GAP 7 resolved**: Automated SL/Target execution is suspended when market is closed. All triggers resume at 09:15 IST.
- **SQLite scope**: Every order placed is appended to `order_audit` table. Every SL/target/trailing state change is checkpointed in `position_checkpoints`.

---

## User Scenarios & Testing

### User Story 1 — SL & Target Initialisation on New Position (Priority: P1)

When a new trade appears, the system instantly computes and stores the SL price and Target price, writes them to the SQLite checkpoint, and broadcasts them to the frontend.

**Independent Test**: Add mock BUY position (entry=1000, qty=50, SL%=2, target%=3 from `user_config`). Verify: `sl_price=980.00`, `target_price=1030.00`, `initial_target_price=1030.00`, `trailing_high_water=0.0`. SQLite checkpoint written. Frontend receives `POSITION_UPDATED` with these values.

**Acceptance Scenarios**:

1. **Given** a new BUY position with `avgEntryPrice=1000`, `qty=50`, `sl_pct=2%, target_pct=3%`, **When** initialised, **Then** `sl_price=980.00`, `target_price=1030.00`, `initial_target_price=1030.00`, `invested_amount=50,000`.
2. **Given** a new SELL position with `avgEntryPrice=500`, `qty=100`, `sl_pct=2%, target_pct=3%`, **When** initialised, **Then** `sl_price=510.00` (above entry), `target_price=485.00` (below entry), `invested_amount=50,000`.
3. **Given** the checkpoint is already in SQLite (recovery scenario), **When** the `PositionMonitor` starts, **Then** it uses the stored `sl_price` / `target_price` / `trailing_high_water` without recalculating.

---

### User Story 2 — Target Hit → Automatic Market Order (Priority: P1)

Price reaches the target. The system places a market exit order within 100ms, logs it to `order_audit`, and removes the position.

**Independent Test**: BUY position `target=1030`; inject tick LTP=1030.1; verify market SELL order placed via mock broker, `order_audit` row written, position status → `SQUAREDOFF`, `POSITION_REMOVED` broadcast.

**Acceptance Scenarios**:

1. **Given** BUY position `target_price=1030`, **When** tick `ltp=1030.0` (≥ target), **Then** system places market SELL order for full qty, writes `TARGET_HIT` to `order_audit`, marks position `SQUARING_OFF`.
2. **Given** SELL position `target_price=485`, **When** `ltp=484.9` (≤ target), **Then** market BUY order placed, `TARGET_HIT` audited.
3. **Given** order is confirmed by Fyers, **When** position reconciliation runs, **Then** position moves to `SQUAREDOFF`, SQLite checkpoint updated, `POSITION_REMOVED` broadcast to frontend.
4. **Given** target already triggered (status=`SQUARING_OFF`), **When** further ticks at or past target arrive, **Then** NO additional orders placed (idempotency via status check).

---

### User Story 3 — SL Hit → Automatic Market Order (Priority: P1)

Price hits the stop-loss. Market exit order placed immediately. The most critical safety mechanism.

**Independent Test**: BUY position `sl=980`; inject tick LTP=979.8; verify market SELL placed, `SL_HIT` audited, status → `SQUARING_OFF` — all within 100ms of tick.

**Acceptance Scenarios**:

1. **Given** BUY position `sl_price=980`, **When** `ltp=980.0` (≤ SL), **Then** market SELL placed, `SL_HIT` logged to `order_audit`.
2. **Given** SELL position `sl_price=510`, **When** `ltp=510.0` (≥ SL), **Then** market BUY placed, `SL_HIT` logged.
3. **Given** SL order in flight (status=`SQUARING_OFF`), **When** more ticks hit SL, **Then** no duplicate orders.
4. **Given** market is closed (after 15:30 IST), **When** a tick passes the SL level, **Then** NO order is placed; a `SL_WOULD_TRIGGER` warning is logged and broadcast to frontend.

---

### User Story 4 — Trailing Stop-Loss (Priority: P1)

For every 0.5% upward move in profit, the SL trails up by 0.5%. SL never moves against the position.

**Table-driven unit test (BUY, entry=1000, initial SL=980, trailing_step=0.5%)**:

| Tick LTP | Profit % | Trailing High-Water | New SL | SL Changed? |
|----------|----------|---------------------|--------|-------------|
| 1000     | 0.0%     | 0.0%                | 980.00 | No          |
| 1003     | 0.3%     | 0.0%                | 980.00 | No (< 0.5%) |
| 1005     | 0.5%     | 0.5%                | 985.00 | ✅ Yes       |
| 1008     | 0.8%     | 0.5%                | 985.00 | No (< 1.0%) |
| 1010     | 1.0%     | 1.0%                | 990.00 | ✅ Yes       |
| 1007     | 0.7%     | 1.0%                | 990.00 | No (price dipped) |
| 1015     | 1.5%     | 1.5%                | 995.00 | ✅ Yes       |
| 994      | —        | —                   | SL_HIT | ✅ Exit      |

**Acceptance Scenarios**:

1. **Given** BUY position, **When** `current_profit_pct` crosses the next 0.5% step above `trailing_high_water`, **Then** `sl_price` increments by `trailing_step × avg_entry_price / 100`, `trailing_high_water` is updated, and the new SL is checkpointed to SQLite.
2. **Given** trailing SL has moved to 995, **When** `ltp` drops to 994, **Then** `SL_HIT` fires.
3. **Given** SELL position, **When** price moves favorably downward, **Then** trailing SL moves down (toward target) by the same step logic.
4. **Given** trailing SL updates, **When** the update is applied, **Then** `POSITION_UPDATED` broadcast to frontend with new `sl_price` AND SQLite `position_checkpoints` is updated.

---

### User Story 5 — Manual: Trail SL to Zero (Breakeven) (Priority: P1)

Trader clicks "Trail SL to Zero" → SL immediately moves to `avgEntryPrice`.

**Acceptance Scenarios**:

1. **Given** BUY position `entry=1000`, current `sl_price=985`, **When** user triggers "Trail SL to Zero", **Then** `sl_price` = 1000, SQLite checkpoint updated, `POSITION_UPDATED` broadcast.
2. **Given** SL is now at breakeven and `ltp` drops to 1000, **Then** `SL_HIT` fires immediately.
3. **Given** market is closed, **When** user triggers "Trail SL to Zero", **Then** action is accepted (just updates the stored SL value — no order placed until market opens and price hits it).

---

### User Story 6 — Manual: Jump Target (Priority: P2)

Trader clicks "Jump Target" → target moves up by 5% of `initial_target_price`.

**Acceptance Scenarios**:

1. **Given** BUY position, `initial_target_price=1030`, `current target=1030`, **When** "Jump Target" pressed, **Then** `target_price = 1030 + (1030 × 0.05) = 1081.50`. SQLite checkpoint updated, `POSITION_UPDATED` broadcast.
2. **Given** pressed again, **When** `target=1081.50`, **Then** `target_price = 1081.50 + (1030 × 0.05) = 1133.00` (increment always = `initial_target_price × 0.05`).
3. **Given** status = `SQUARING_OFF`, **When** "Jump Target" pressed, **Then** API returns 409 Conflict: "Position is being squared off."

---

### User Story 7 — Manual: Square Off Individual Position (Priority: P1)

Trader clicks "Square Off" on a trade → immediate market exit order.

**Acceptance Scenarios**:

1. **Given** BUY position is OPEN, **When** "Square Off" API call received, **Then** market SELL order placed, `MANUAL_SQUARE_OFF` written to `order_audit`, status → `SQUARING_OFF`.
2. **Given** order fails (Fyers API error), **Then** status reverts to `OPEN`, `ORDER_FAILED` event broadcast with error details, position flagged `NEEDS_MANUAL_REVIEW`.

---

### User Story 8 — EXIT ALL with Semaphore Rate Control (Priority: P1)

All open positions exited with max 3 concurrent order requests at a time.

**Acceptance Scenarios**:

1. **Given** 9 open positions, **When** EXIT ALL triggered, **Then** orders are fired in batches of 3 concurrent requests (semaphore); all 9 positions eventually reach `SQUARING_OFF`.
2. **Given** one order in the batch of 3 fails, **When** detected, **Then** the failed position is flagged `NEEDS_MANUAL_REVIEW`; the other 8 proceed normally.
3. **Given** EXIT ALL in progress, **When** a new "Square Off" arrives for one of the positions, **Then** it is ignored (position already in `SQUARING_OFF` state — idempotency).

---

### Edge Cases

- SL price == target price (invalid config): reject at initialisation, log error, notify frontend, do not apply any rules.
- Market gaps through SL on open: first tick after open checked immediately; if past SL, exit order placed instantly.
- Order placement fails on SL hit: status set to `NEEDS_MANUAL_REVIEW`, `ORDER_FAILED` broadcast to frontend, NO retry (safety — human must intervene).
- Two concurrent ticks for same position trigger SL simultaneously: atomic compare-and-swap on `PositionStatus` (only first caller transitions `OPEN → SQUARING_OFF`; subsequent calls see `SQUARING_OFF` and skip).
- User presses "Trail SL to Zero" when price is already below entry (in loss): SL still set to entry price; position exits at next tick below entry, cutting the loss at breakeven.

---

## Requirements

### Functional Requirements

- **FR-001**: On new position, compute `sl_price`, `target_price`, `initial_target_price` using `user_config` values; write to SQLite `position_checkpoints` and broadcast `POSITION_UPDATED`.
- **FR-002**: BUY: `sl = entry × (1 - sl_pct/100)`, `target = entry × (1 + target_pct/100)`. SELL: `sl = entry × (1 + sl_pct/100)`, `target = entry × (1 - target_pct/100)`.
- **FR-003**: On each tick (delivered via EventBus from Spec 3), if market is OPEN: evaluate SL and Target conditions for the affected position.
- **FR-004**: Trailing SL step: when `current_profit_pct ≥ trailing_high_water + trailing_step_pct`, increment `sl_price` by `(trailing_step_pct / 100) × avg_entry_price` in favorable direction; update `trailing_high_water`; write checkpoint to SQLite.
- **FR-005**: On SL or Target hit: atomically transition status `OPEN → SQUARING_OFF`. Place market order via `fyers-client` thin client (`POST /api/v3/orders/sync`, `type=2 MARKET`). Write `order_audit` row immediately (pre-order).
- **FR-006**: Order audit record includes: `position_id`, `symbol`, `side`, `qty`, `trigger_type`, `trigger_price`, `ltp_at_trigger`, `broker_order_id`, `broker_response`, `status (PENDING→SUCCESS/FAILED)`, `placed_at`.
- **FR-007**: If market is CLOSED when SL/Target condition is met: log `SL_WOULD_TRIGGER` / `TARGET_WOULD_TRIGGER` warning, broadcast to frontend, do NOT place order.
- **FR-008**: REST endpoints for manual overrides:
  - `POST /api/positions/{id}/square-off` — market exit, `MANUAL_SQUARE_OFF` audited
  - `POST /api/positions/{id}/trail-sl-to-zero` — `sl_price = avg_entry_price`, checkpoint written
  - `POST /api/positions/{id}/jump-target` — `target_price += initial_target_price × 0.05`, checkpoint written
  - `POST /api/positions/exit-all` — semaphore(3) concurrent market orders across all OPEN positions
- **FR-009**: All manual override endpoints: validate session, validate position is OPEN, respond ≤ 500ms.
- **FR-010**: Semaphore for EXIT ALL: `tokio::sync::Semaphore::new(3)` — max 3 concurrent Fyers order placement requests.
- **FR-011**: Trailing step `trailing_step_pct` read from SQLite `user_config`; default = 0.5%.

### Key Entities

- **TrailingState** (embedded in PositionState): `{ trailing_high_water: f64, trailing_step_pct: f64 }`.
- **TriggerType**: `SL_HIT | TARGET_HIT | MANUAL_SQUARE_OFF | EXIT_ALL | SL_WOULD_TRIGGER | TARGET_WOULD_TRIGGER`.
- **SQLite `order_audit`**: Immutable append-only log — never updated after insertion (only `status` field updated on broker response).

---

## Success Criteria

- **SC-001**: Tick → SL/Target evaluation → order placement API call ≤ 100ms (integration test with mock broker, measured with `tokio::time::Instant`).
- **SC-002**: Zero duplicate orders across 10,000 simulated tick events in concurrent load test (idempotency verified).
- **SC-003**: Trailing SL math correct for all 50 table-driven unit test cases including BUY/SELL, varying steps, price reversals.
- **SC-004**: EXIT ALL with 15 positions (semaphore=3) completes with no more than 5 concurrent HTTP requests to Fyers at any instant (verified with mock broker call counting).
- **SC-005**: Every order placed results in a `order_audit` row within 100ms (tested with in-memory SQLite).
- **SC-006**: After a backend restart with 3 open positions in SQLite checkpoints, all 3 `PositionMonitor` tasks correctly resume with stored SL/target/trailing state and respond to the next injected tick correctly.

---

## Assumptions

- Fyers market order params: `{ symbol, qty, type: 2, side: 1 (BUY) or -1 (SELL), productType: "INTRADAY", limitPrice: 0, stopPrice: 0, validity: "DAY", disclosedQty: 0, offlineOrder: false }`.
- SL and Target are tracked in application memory + SQLite only — they are NOT bracket orders on Fyers. The app fires market orders when conditions are met.
- Jump Target increment always references `initial_target_price` (frozen at position creation) — not the current target price.
- Trailing step default = 0.5%; configurable via `user_config` (applied globally, not per-position).
- Market hours determination: `chrono` crate with IST timezone (`Asia/Kolkata`). Weekends and holidays not specifically handled in v1 (system operates Mon–Fri, no holiday calendar).
