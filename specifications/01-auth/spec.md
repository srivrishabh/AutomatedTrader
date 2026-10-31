# Feature Specification: Authentication & Session Management

**Feature Branch**: `01-auth-session`

**Created**: 2026-10-31 | **Revised**: 2026-10-31 (v2 — gap resolutions applied)

**Status**: Draft

---

## Gap Resolutions Applied
- **GAP 5 resolved**: Re-login modal shown in-app when token expires; `access_token` + `client_id` + `expiry_timestamp` persisted in SQLite and loaded on startup to survive restarts.
- **SQLite scope**: Token store is one of four SQLite concerns (tokens, user_config, position_checkpoints, order_audit).

---

## User Scenarios & Testing

### User Story 1 — Silent Login on Restart (via SQLite Token) (Priority: P1)

The trader restarts the backend at 9:00 AM. The system reads the SQLite token store, finds a valid (non-expired) `access_token` from yesterday's session, and the frontend immediately shows the dashboard — no re-auth needed.

**Why this priority**: Eliminates the daily friction of re-authentication if the token is still valid. Provides fault-tolerant session recovery.

**Independent Test**: Pre-populate SQLite `tokens` table with a valid unexpired token, restart the backend, call `GET /api/auth/status` — expect `{ authenticated: true, clientId: "XY12345" }` with no OAuth flow triggered.

**Acceptance Scenarios**:

1. **Given** SQLite contains a token with `expiry_timestamp > now()`, **When** the backend starts, **Then** it loads the token, sets the in-memory session as authenticated, and `GET /api/auth/status` returns `{ authenticated: true }`.
2. **Given** SQLite contains a token with `expiry_timestamp <= now()` (expired), **When** the backend starts, **Then** it ignores the stale token, session is unauthenticated, and the frontend receives `{ authenticated: false }`.
3. **Given** SQLite has no token row at all (first launch), **When** the backend starts, **Then** session is unauthenticated and the user is shown the login page.

---

### User Story 2 — Morning Login via Fyers OAuth (Priority: P1)

The trader opens the app and finds the login page (token expired or first use). They click "Login with Fyers", complete the OAuth + TOTP flow, and are redirected to the dashboard. The new token is immediately persisted to SQLite.

**Why this priority**: Foundation — everything depends on this working.

**Independent Test**: Mock the Fyers token exchange endpoint to return a valid `access_token`; verify the backend writes it to SQLite and `GET /api/auth/status` returns authenticated.

**Acceptance Scenarios**:

1. **Given** the user is unauthenticated, **When** they visit `/`, **Then** they see the Login page with a "Login with Fyers" button and a status line "No active session found."
2. **Given** the user clicks "Login with Fyers", **When** the backend generates the Fyers OAuth URL (with `client_id`, `redirect_uri`, `response_type=code`, random `state`), **Then** the frontend redirects the browser to that URL.
3. **Given** the user completes Fyers authentication (TOTP + PIN), **When** Fyers redirects back with `auth_code`, **Then** the backend: validates `state`, calls `POST /api/v3/validate-authcode`, receives `access_token`, writes `{ access_token, client_id, expiry_timestamp: now() + 24h }` to SQLite `tokens` table, sets in-memory session.
4. **Given** login succeeds, **When** the user lands on `/auth/callback`, **Then** the React app auto-redirects to `/dashboard` within 1 second.

---

### User Story 3 — In-App Session Expiry & Re-authentication (Priority: P1)

The token expires mid-session (e.g., the trader forgot to log out and it's the next day). The app detects the expiry and shows a re-login modal — no page refresh or data loss.

**Why this priority**: Happens every trading day if the app is kept open; must be handled gracefully.

**Independent Test**: Replace the in-memory token with an expired one, make any Fyers API call, verify the backend emits `SESSION_EXPIRED` via WebSocket and the frontend shows a re-login modal within 2 seconds.

**Acceptance Scenarios**:

1. **Given** the system makes a Fyers API call and receives HTTP 401, **When** this is detected, **Then** the backend marks the session expired, invalidates SQLite token row, and emits `{"type":"SESSION_EXPIRED"}` via the frontend WebSocket.
2. **Given** the frontend receives `SESSION_EXPIRED`, **When** displayed, **Then** a non-dismissible re-login modal appears with "Your Fyers session has expired. Please log in again." and a "Login with Fyers" button.
3. **Given** the user completes re-authentication via the modal, **When** the new token is written to SQLite, **Then** all active WebSocket subscriptions are re-established automatically without restarting the app.
4. **Given** re-auth completes, **When** the session is restored, **Then** all position monitors resume from their last persisted checkpoint state (no position state is lost).

---

### User Story 4 — Logout (Priority: P3)

The trader clicks "Logout" at end of day. The SQLite token is deleted, the in-memory session is cleared, and they are returned to the login page.

**Acceptance Scenarios**:

1. **Given** the user is logged in, **When** they click "Logout", **Then** the backend deletes the `tokens` row from SQLite, destroys the in-memory session, and the frontend redirects to the login page.
2. **Given** logged out, **When** any protected endpoint is called, **Then** the backend returns HTTP 401.

---

### Edge Cases

- OAuth `state` mismatch on callback → reject with 400, log security event, show error page, do NOT write to SQLite.
- Fyers token exchange network failure → show "Login failed — please try again" error; no partial write to SQLite.
- Two browser tabs open → both share the same server-side session; re-auth in one tab triggers `SESSION_RESTORED` WebSocket event to all tabs.
- Backend restarts mid-trading-day → SQLite token is valid, session is restored, position checkpoints are replayed (handled in Spec 2).
- SQLite write failure on token save → log critical error, keep session in-memory but warn frontend: "Session not persisted — restart may require re-login."

---

## Requirements

### Functional Requirements

- **FR-001**: On startup, system MUST read the SQLite `tokens` table; if a valid non-expired token exists, establish the in-memory session without requiring OAuth.
- **FR-002**: System MUST expose `GET /api/auth/login` returning the Fyers OAuth URL with `client_id`, `redirect_uri`, `response_type=code`, and a cryptographically random `state` stored in a short-lived (5-min) in-memory map.
- **FR-003**: System MUST expose `GET /api/auth/callback` that validates `state`, exchanges `auth_code` for `access_token` via Fyers `POST /api/v3/validate-authcode`, and persists `{ access_token, client_id, expiry_timestamp }` to SQLite atomically.
- **FR-004**: System MUST expose `GET /api/auth/status` returning `{ authenticated: bool, clientId: string | null, expiresAt: string | null }`.
- **FR-005**: System MUST expose `POST /api/auth/logout` that deletes the SQLite token row and destroys the in-memory session.
- **FR-006**: All non-auth API endpoints MUST be protected by authentication middleware returning HTTP 401 if no valid session.
- **FR-007**: The `access_token` MUST never be transmitted to the frontend — it remains server-side only (SQLite + memory).
- **FR-008**: Any Fyers API 401 response MUST trigger: session invalidation, SQLite token deletion, and a `SESSION_EXPIRED` WebSocket event to all connected frontend clients.
- **FR-009**: After re-authentication, system MUST emit `SESSION_RESTORED` WebSocket event and resume all position monitors from SQLite checkpoints.
- **FR-010**: `expiry_timestamp` stored as UTC Unix timestamp (seconds). Default TTL = 86400 seconds (24h) from token issuance.

### Key Entities

- **TokenRecord** (SQLite `tokens`): `{ id INTEGER PK, access_token TEXT, client_id TEXT, expiry_timestamp INTEGER, created_at INTEGER }`.
- **Session** (in-memory): `{ access_token: String, client_id: String, expires_at: DateTime<Utc> }` — held in `Arc<RwLock<Option<Session>>>`.
- **AuthState** (in-memory, 5-min TTL): `HashMap<state_nonce, created_at>` — prevents CSRF on OAuth callback.

---

## Success Criteria

- **SC-001**: Silent startup re-auth (valid SQLite token) completes in under 500ms.
- **SC-002**: Full OAuth login flow (click → Fyers → callback → dashboard) completes under 90 seconds including user TOTP input.
- **SC-003**: Token is persisted to SQLite within 200ms of the callback endpoint being hit.
- **SC-004**: `SESSION_EXPIRED` WebSocket event reaches all frontend clients within 2 seconds of a Fyers 401 response.
- **SC-005**: 100% of protected endpoints return HTTP 401 without a valid session (integration test verified).
- **SC-006**: Zero `access_token` values appear in any HTTP response body, browser storage, or WebSocket message.

---

## Assumptions

- Fyers OAuth requires browser interaction (TOTP/PIN) — cannot be automated beyond the re-login modal pattern.
- `redirect_uri` = `http://localhost:8080/api/auth/callback` for dev; configurable via `FYERS_REDIRECT_URI` env var.
- `FYERS_APP_ID` and `FYERS_SECRET_KEY` are provided via environment variables — never stored in SQLite or transmitted to frontend.
- Single-user application — one active token at a time.
- Fyers `access_token` TTL is 24 hours from issuance (confirmed from Fyers docs). The `expiry_timestamp` in SQLite is set conservatively as `issued_at + 23h` to account for clock skew.
- SQLite database file: `~/.automatedtrader/data.db` (path configurable via `DB_PATH` env var).
