# Feature Specification: React Frontend Dashboard

**Feature Branch**: `05-react-dashboard`

**Created**: 2026-10-31 | **Revised**: 2026-10-31 (v2 — gap resolutions applied)

**Status**: Draft

---

## Gap Resolutions Applied
- **GAP 1 resolved**: No PCR column in the Indices section — only live price, change amount, and change %.
- **GAP 5 resolved**: Re-login modal shown in-app when `SESSION_EXPIRED` WebSocket event is received — no full page reload.
- **GAP 7 resolved**: "Market Closed" banner shown outside 09:15–15:30 IST; all order buttons disabled; stale indicator on position rows.

---

## User Scenarios & Testing

### User Story 1 — Login Page & Silent Auth Recovery (Priority: P1)

On app open, if the backend has a valid SQLite token, the user goes directly to the dashboard. If not, they see the login page.

**Independent Test**: Mock `GET /api/auth/status` returning `{ authenticated: true }` — verify React Router redirects to `/dashboard` without showing the login page.

**Acceptance Scenarios**:

1. **Given** `GET /api/auth/status` returns `{ authenticated: true }`, **When** the app loads at `/`, **Then** the user is immediately redirected to `/dashboard`.
2. **Given** `GET /api/auth/status` returns `{ authenticated: false }`, **When** the app loads, **Then** the Login page is shown with "Login with Fyers" button and "No active session found" subtitle.
3. **Given** the user clicks "Login with Fyers", **When** auth URL is fetched from backend, **Then** the browser navigates to the Fyers OAuth portal.
4. **Given** Fyers redirects back to `/auth/callback`, **When** the callback component mounts, **Then** it polls `GET /api/auth/status` until authenticated then navigates to `/dashboard`.

---

### User Story 2 — Market Closed Banner (Priority: P1)

Outside market hours (before 09:15 or after 15:30 IST), the dashboard clearly shows a "Market Closed" state.

**Independent Test**: Mock a `MARKET_CLOSED` WebSocket event, verify: banner appears, all order buttons have `disabled` attribute, position rows show a "Stale" badge.

**Acceptance Scenarios**:

1. **Given** the frontend receives `{"type":"MARKET_CLOSED"}` from the backend, **When** displayed, **Then** a persistent amber banner "⚠ Market Closed — Automated trading suspended" appears at the top of the page.
2. **Given** market is closed, **When** position rows are rendered, **Then** each row shows a grey "Stale" badge next to the P/L value.
3. **Given** market is closed, **When** buttons are rendered, **Then** Square Off, Trail SL to Zero, Jump Target, and EXIT ALL are all visually disabled and non-clickable.
4. **Given** the frontend receives `{"type":"MARKET_OPEN"}` at 09:15, **When** displayed, **Then** the banner disappears and all buttons become active.

---

### User Story 3 — Indices Bar (Priority: P1)

The top bar shows Nifty 50 and Sensex with live price, change amount and %, updating in real-time. No PCR field.

**Acceptance Scenarios**:

1. **Given** an `INDEX_UPDATE` for Nifty arrives, **When** rendered, **Then** the Nifty card shows: "NIFTY 50", LTP formatted as `24,250.75`, change as `▲ 125.30 (0.52%)` in green.
2. **Given** Sensex declines, **When** `INDEX_UPDATE` arrives, **Then** Sensex card shows: `▼ -234.10 (-0.62%)` in red.
3. **Given** the data feed is reconnecting, **When** displayed, **Then** indices show last known values with a small "●" reconnecting indicator (no values zeroed out).
4. **Given** market is closed, **When** displayed, **Then** indices show last known value with "Market Closed" label (not "--").

---

### User Story 4 — Trades Grid with Live Updates (Priority: P1)

The main section shows all open positions in a grid, updating live as ticks arrive.

**Acceptance Scenarios**:

1. **Given** 3 positions are received via `POSITIONS_SNAPSHOT`, **When** rendered, **Then** the grid shows 3 rows each with: Instrument, Side badge (BUY green / SELL red), Qty, Invested (₹), Avg Entry, LTP, P/L (₹), P/L (%), SL Value, Target Value.
2. **Given** a `TICK_UPDATE` arrives for one position, **When** the P/L cells update, **Then** a brief flash animation plays: green pulse for profit increase, red pulse for loss increase.
3. **Given** `POSITION_ADDED` arrives, **When** rendered, **Then** a new row slides into the grid at the top (most recent first) with a brief highlight animation.
4. **Given** `POSITION_REMOVED` arrives, **When** rendered, **Then** the corresponding row fades out over 500ms then is removed.
5. **Given** `POSITION_UPDATED` arrives (SL or target changed), **When** rendered, **Then** only the SL Value or Target Value cell updates — no full row re-render.

---

### User Story 5 — Per-Row Action Buttons (Priority: P1)

Each position row has three action buttons. They are disabled when market is closed.

**Acceptance Scenarios**:

1. **Given** market is OPEN and a row is shown, **When** user clicks `[Square Off]`, **Then** a confirmation dialog appears: "Square off {instrument}? This places a market order and cannot be undone." with Cancel / Confirm (danger red).
2. **Given** user confirms Square Off, **When** API call is in flight, **Then** the row's buttons show a spinner; the row shows "Squaring off…" status.
3. **Given** Square Off API succeeds, **When** `POSITION_REMOVED` arrives via WebSocket, **Then** the row fades out.
4. **Given** user clicks `[Trail SL to Zero]`, **When** API call succeeds, **Then** the SL Value cell immediately updates to the entry price (optimistic update) and a success toast "SL moved to breakeven for {instrument}" appears.
5. **Given** user clicks `[Jump Target]`, **When** API call succeeds, **Then** the Target Value cell updates to the new higher value with a brief green pulse and a toast "Target raised for {instrument}".
6. **Given** any action API call fails, **When** error response arrives, **Then** a red error toast "{Action} failed: {error_message}" is shown for 5 seconds; button state reverts.

---

### User Story 6 — EXIT ALL Button (Priority: P1)

Global panic button exits all positions. Shows progress, respects market hours.

**Acceptance Scenarios**:

1. **Given** market is OPEN with 5 positions, **When** user clicks `[EXIT ALL]`, **Then** a modal appears: "EXIT ALL POSITIONS" title, "This will immediately exit all 5 open positions at market price. THIS CANNOT BE UNDONE." body, a red "Confirm Exit All" button and "Cancel".
2. **Given** user confirms, **When** in progress, **Then** EXIT ALL button is replaced with a progress indicator "Exiting 5 positions…"; each position row shows "Squaring off…" state.
3. **Given** all positions exit, **When** grid is empty, **Then** the grid shows "All positions exited" and EXIT ALL button returns to its normal state.
4. **Given** market is CLOSED, **When** EXIT ALL is rendered, **Then** button is disabled and shows tooltip "Market is closed".

---

### User Story 7 — In-App Re-login Modal (Priority: P1)

When `SESSION_EXPIRED` arrives via WebSocket, a non-dismissible modal prompts re-auth without losing app state.

**Acceptance Scenarios**:

1. **Given** `SESSION_EXPIRED` WebSocket event arrives, **When** displayed, **Then** a full-screen modal overlay appears: "Session Expired — Please log in again" with a "Login with Fyers" button. The modal cannot be closed by clicking outside or pressing Escape.
2. **Given** the user clicks "Login with Fyers" in the modal, **When** they complete Fyers OAuth, **Then** the modal closes automatically and the dashboard shows the current position state (no page reload).
3. **Given** re-auth completes and `SESSION_RESTORED` is received, **When** displayed, **Then** a success toast "Session restored — trading resumed" appears and all position monitors resume.

---

### User Story 8 — Configuration Panel (Priority: P2)

User sets default SL%, Target%, and Trailing Step% which are persisted to backend SQLite.

**Acceptance Scenarios**:

1. **Given** the config panel is visible, **When** user changes "Default SL %: [2.0]" to 1.5 and clicks Apply, **Then** a `PUT /api/config` call is made with `{ defaultSlPct: 1.5 }` and a toast "Config saved" appears.
2. **Given** new defaults are saved, **When** the next new position is detected, **Then** it uses the new `sl_pct=1.5%` (not 2.0%).
3. **Given** user changes config, **When** displayed in config panel, **Then** current values are always loaded from `GET /api/config` on dashboard mount (not from local state).

---

### Edge Cases

- Malformed WebSocket JSON → log to console, discard, no crash.
- 50+ positions grid at high tick frequency → AG Grid virtualised rows handle this; frontend debounce (100ms) limits re-renders.
- Browser tab is backgrounded → WebSocket stays connected; updates buffer and apply when tab regains focus.
- Mobile viewports (< 768px) → grid columns reduce to essential: Instrument, P/L%, SL, Target, action button (more in expandable drawer).

---

## Requirements

### Functional Requirements

- **FR-001**: App MUST use React 18 + TypeScript + Vite. Routing via React Router v6 with protected routes (redirect to login if `authenticated: false`).
- **FR-002**: On mount, app MUST call `GET /api/auth/status`; if authenticated, navigate to `/dashboard`; else render Login page.
- **FR-003**: Login page calls `GET /api/auth/login` to fetch Fyers auth URL and navigates the browser to it.
- **FR-004**: `/auth/callback` route polls `GET /api/auth/status` (max 10 attempts, 1s apart) until `authenticated: true`, then navigates to `/dashboard`.
- **FR-005**: Dashboard establishes a WebSocket to `ws://{host}/ws` on mount. Handles all event types: `POSITIONS_SNAPSHOT`, `POSITION_ADDED`, `POSITION_UPDATED`, `POSITION_REMOVED`, `TICK_UPDATE`, `INDEX_UPDATE`, `SESSION_EXPIRED`, `SESSION_RESTORED`, `MARKET_OPEN`, `MARKET_CLOSED`, `ORDER_FAILED`.
- **FR-006**: On `SESSION_EXPIRED`: show non-dismissible re-login modal. On `SESSION_RESTORED`: close modal, show toast, resume.
- **FR-007**: On `MARKET_CLOSED`: show amber banner, disable all order buttons, show "Stale" badge on position rows. On `MARKET_OPEN`: remove banner, enable buttons.
- **FR-008**: Indices bar shows 2 cards (NIFTY 50, SENSEX) with: LTP, change (±₹), change %, colored by direction. **No PCR field**.
- **FR-009**: Trades grid uses AG Grid Community with columns: Instrument, Side, Qty, Invested (₹), Avg Entry, LTP, P/L (₹), P/L %, SL Value, Target Value, Actions. P/L cells use `profit`/`loss` CSS classes + flash animation on update.
- **FR-010**: Each row's Actions cell: `[Square Off]` (with confirmation dialog), `[Trail SL to Zero]` (with success toast), `[Jump Target]` (with success toast). All disabled when market closed.
- **FR-011**: EXIT ALL button in page header: red/danger styling, confirmation modal with count of open positions, progress state, disabled when market closed.
- **FR-012**: Config panel: number inputs for `defaultSlPct` (step=0.1, min=0.1, max=20), `defaultTargetPct` (step=0.1), `trailingStepPct` (step=0.1). Values loaded from `GET /api/config`, saved to `PUT /api/config`.
- **FR-013**: State management via Zustand stores: `usePositionStore`, `useIndexStore`, `useWsStore` (connection state), `useMarketStore` (open/closed), `useConfigStore`.
- **FR-014**: All order action calls use Axios to backend REST; auth is httpOnly session cookie (no manual token management in frontend).
- **FR-015**: App must be responsive: full grid at ≥ 1024px; reduced columns with expandable row detail at < 1024px.

### Key Entities (Frontend State)

- **Position (UI)**: `{ id, symbol, displayName, side, qty, investedAmount, avgEntry, ltp, pl, plPct, slPrice, targetPrice, status, isStale }`.
- **IndexData**: `{ symbol, displayName, ltp, change, changePct, isStale }`. **No `pcr` field**.
- **WsConnectionState**: `CONNECTED | RECONNECTING | DISCONNECTED`.
- **MarketState**: `OPEN | CLOSED | UNKNOWN`.
- **UserConfig**: `{ defaultSlPct, defaultTargetPct, trailingStepPct }`.

---

## Success Criteria

- **SC-001**: Dashboard renders initial positions within 2 seconds of page load (post-auth).
- **SC-002**: P/L flash animation visible within 200ms of `TICK_UPDATE` arriving at browser.
- **SC-003**: "Market Closed" banner appears within 1 second of `MARKET_CLOSED` WebSocket event.
- **SC-004**: Re-login modal appears within 1 second of `SESSION_EXPIRED` event.
- **SC-005**: App renders correctly at 768px, 1024px, 1440px (visual regression tested with Playwright screenshots).
- **SC-006**: All action buttons show loading state during API call; no double-submission possible (buttons disabled while request is in-flight).
- **SC-007**: AG Grid with 50 rows updating at 10 ticks/second maintains ≥ 30fps (Chrome Performance profiler).

---

## Assumptions

- Backend WebSocket: `ws://localhost:8080/ws` in dev; `VITE_WS_URL` env var for production.
- Backend REST: `http://localhost:8080` in dev; `VITE_API_URL` env var for production.
- Auth managed via httpOnly session cookie from backend — no JWT in localStorage.
- AG Grid Community Edition — free, no Enterprise license needed.
- Frontend is single-user desktop/tablet app (Chrome, Firefox, Edge latest).
- In production, the React build (`dist/`) is served as static files by the Rust actix-web server from the same origin (no CORS needed).
- `displayName` for instruments: derived from symbol by stripping exchange prefix (e.g., `NSE:RELIANCE-EQ` → `RELIANCE-EQ`). Full display name mapping is a v2 enhancement.
