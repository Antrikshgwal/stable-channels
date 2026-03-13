# Stable Channels LSP Production Hardening Plan

## Phase 1: Stabilize the LSP (Weeks 1–3)

### 1.1 Health & Observability
- **Add `/health` endpoint** to `src/bin/lsp_backend.rs` — returns JSON with: LDK node status, peer count, SQLite connectivity, latest price age, uptime. This is the first thing ops tooling (Umbrel, Docker) will probe.
- **Add `/metrics` endpoint** (Prometheus text format) — expose counters/gauges: active channels, total capacity sats, stability payments sent/received, price fetch failures, payment failures, peer disconnects. Use the `prometheus` crate (lightweight, no runtime dependency).
- **Structured logging** — replace ad-hoc `println!`/`eprintln!` with `tracing` crate. Add `tracing-subscriber` with JSON formatter for production and pretty formatter for dev. Instrument key functions with `#[instrument]` spans: stability loop iterations, payment sends, price fetches, API handlers.

### 1.2 API Authentication & Security
- **API key authentication** — add Axum middleware that checks `Authorization: Bearer <key>` header against a config value (`LSP_API_KEY` env var). Apply to all `/api/*` routes. The mobile app only uses `/api/register-push`, so that single route can be exempted or use a separate key.
- **Rate limiting** — add `tower::limit::RateLimitLayer` to the Axum router (e.g., 60 req/min per IP). Protects against abuse on public-facing endpoints.
- **Input validation** — audit all `POST` handlers in `lsp_backend.rs` (lines 584–600) for missing validation: `close_channel` channel_id format, `pay` invoice parsing, `onchain_send` address + amount bounds, `edit_stable_channel` target_usd range.

### 1.3 Stability Loop Hardening
- **Concurrent channel safety** — the stability loop in `lsp_backend.rs` iterates channels sequentially. Audit for races between the stability loop and incoming API requests that modify `StableChannel` state (e.g., `edit_stable_channel` while a stability payment is in flight). Add per-channel locking or use a channel-keyed `DashMap` instead of `Mutex<Vec<StableChannel>>`.
- **Price feed resilience** — `price_feeds.rs` fetches sequentially and uses the last successful value. Improve: fetch all 5 in parallel with `tokio::join!`, take the median of successful results, reject if fewer than 3 feeds respond. Log which feeds failed. Add a staleness circuit breaker: if cached price is >60s old, pause stability payments and log an alert.
- **Payment failure recovery** — when a stability keysend fails (PaymentFailed event in `user.rs` ~line 1897), the current code logs it but doesn't retry or adjust state. Add: retry with exponential backoff (max 3 attempts), and if all fail, mark the channel's risk_level and log an audit event so the operator can intervene.
- **Force-close handling** — ensure `ChannelClosed` event properly reconciles the `StableChannel` in the DB (mark closed, record final balances, stop stability checks for that channel). Verify the channel is removed from the active stability loop.

### 1.4 Expanded Test Coverage
- **Unit tests for `stable.rs`** — test `check_stability()` with: no drift (no payment), small drift below threshold (no payment), drift above threshold (correct payment amount and direction), cooldown period (no payment even with drift), edge cases (zero balance, very large drift).
- **Unit tests for `price_feeds.rs`** — test median calculation with: all feeds ok, 2 feeds down, all feeds down (circuit breaker), stale cache.
- **API integration tests** — use `axum::test` helpers to test each endpoint: `/health` returns 200, `/api/balance` returns valid JSON, `/api/channels` with no channels returns empty array, auth rejection on missing/bad API key.
- **Multi-channel regtest test** — extend `tests/regtest.rs`: open 2+ stable channels with different `expected_usd`, trigger price change, verify both channels get independent stability payments.

### 1.5 Configuration Externalization
- **Move hardcoded constants to env vars / config file** — `constants.rs` has hardcoded LSP pubkey, IP address (`34.198.44.89`), ports, network selection. Create a `config.rs` module that reads from env vars with sensible defaults. Key configs: `LSP_BIND_ADDR`, `LSP_PORT`, `LSP_NETWORK` (testnet/mainnet), `LSP_DATA_DIR`, `LSP_API_KEY`, `STABILITY_INTERVAL_SECS`, `STABILITY_THRESHOLD_PCT`, `PRICE_CACHE_TTL_SECS`.

---

## Phase 2: Web-Based Operator Dashboard (Weeks 4–6)

### 2.1 Backend API Extensions
Add new endpoints to `lsp_backend.rs` for dashboard needs (the existing Egui frontend stays as-is):

- `GET /api/dashboard/summary` — aggregated view: total channels, total capacity, total USD exposure, total native BTC, number of peers online, last stability loop timestamp, system uptime.
- `GET /api/dashboard/channel/{id}/history` — historical stability payments for a specific channel (query `payments` table filtered by channel).
- `GET /api/dashboard/pnl` — profit/loss: total fees earned, total stability payments made/received, net position.
- `GET /api/dashboard/alerts` — list of active alerts: channels with high drift, offline peers, stale price, failed payments in last hour.
- `GET /api/dashboard/price_history?range=24h` — price data for charting (query `price_history` table).

### 2.2 Static Web UI (served from Axum)
- **Serve static files** — add a `static/` directory at repo root. Configure Axum to serve it at `/dashboard/*` using `tower-http::services::ServeDir`.
- **Single-page layout** — `static/index.html` with:
  - **Header bar**: LSP node ID (truncated), network (testnet/mainnet), uptime, current BTC price
  - **Summary cards**: Total channels, total capacity (BTC + USD), total USD exposure, peers online/total
  - **Channels table**: sortable by capacity, drift %, status. Each row: channel ID (truncated), peer pubkey, expected_usd, current_usd, drift %, backing_sats, native_sats, is_stable, online status. Click to expand → payment history, edit target_usd, close button.
  - **Stability chart**: live-updating line chart of BTC price + per-channel drift over time (use lightweight Chart.js or similar via CDN)
  - **Alerts panel**: color-coded list of active issues
  - **Audit log**: scrollable, filterable log viewer (tail last 200 lines, auto-refresh)
- **JavaScript** — `static/dashboard.js`: fetch from `/api/*` endpoints every 5s (or use SSE for real-time), render into DOM. No build step, no framework. ~500 lines.
- **CSS** — `static/dashboard.css`: dark theme (matches Bitcoin/Lightning aesthetic), responsive layout with CSS grid. ~200 lines.
- **Auth** — dashboard routes behind the same API key middleware (prompt for key on load, store in sessionStorage).

### 2.3 Server-Sent Events (Optional Enhancement)
- Add `GET /api/events` SSE endpoint for real-time push of: new stability payments, channel opens/closes, price updates, alerts. This avoids polling and gives the dashboard instant reactivity. Use `axum::response::Sse` with `tokio::sync::broadcast`.

---

## Phase 3: Umbrel App Packaging (Weeks 7–8)

### 3.1 Dockerization
- **Dockerfile** — multi-stage build:
  1. Builder stage: `rust:1.77-slim` → `cargo build --release --bin lsp_backend`
  2. Runtime stage: `debian:bookworm-slim` → copy binary + static assets + migration scripts
  3. Expose ports: 9735 (Lightning P2P), 8080 (HTTP API + dashboard)
  4. Entrypoint: the `lsp_backend` binary with env var configuration
- **docker-compose.yml** — full stack:
  - `lsp` service: the Stable Channels LSP
  - `bitcoind` service: Bitcoin Core (or connect to Umbrel's existing bitcoind)
  - `electrs` service: Electrum server (or Umbrel's existing electrs)
  - Shared volume for LDK data persistence
  - Environment file for configuration

### 3.2 Umbrel App Manifest
- Create `umbrel-app/` directory with Umbrel app structure:
  - `umbrel-app.yml` — app metadata (name, version, description, icon, port, dependencies: bitcoind, electrs)
  - `docker-compose.yml` — adapted from 3.1, using Umbrel's networking conventions (`APP_BITCOIN_NODE_IP`, `APP_ELECTRS_NODE_IP` env vars to connect to existing Umbrel services)
  - `exports.sh` — expose LSP connection info (node pubkey, Lightning port, dashboard URL)
- **Data persistence** — use Umbrel's app data directory convention for LDK node data + SQLite DB
- **Dependency wiring** — Umbrel runs bitcoind and electrs already; the LSP app should discover and connect to them via Umbrel's internal DNS/env vars rather than bundling its own

### 3.3 Testing the Umbrel Package
- Test on Umbrel OS (or umbrel-dev docker environment):
  1. Install app from local app store
  2. Verify LSP starts, connects to bitcoind/electrs, generates node keys
  3. Verify dashboard accessible at `http://umbrel.local:<port>`
  4. Open a channel from a mobile wallet, verify stability loop runs
  5. Document any Umbrel-specific configuration needed

---

## Phase 4: Demo Video & Documentation (Week 9)

### 4.1 Demo Script
Record a walkthrough showing:
1. **Umbrel install** — one-click install of Stable Channels LSP app on Umbrel
2. **Dashboard tour** — open web dashboard, show channel summary, price chart, empty state
3. **Mobile connect** — open Stable Channels mobile app, connect to LSP, open a stable channel
4. **Stability in action** — show price movement triggering stability payments (use testnet + mock price or wait for real movement). Show drift % changing on dashboard, payments appearing in audit log.
5. **Operator actions** — edit channel target USD, view P&L, check alerts, close a channel
6. **End state** — show the full system healthy: multiple channels, stability payments flowing, dashboard green

### 4.2 Documentation Updates
- Update `README.md` with:
  - Architecture diagram (app ↔ LSP ↔ bitcoind)
  - Quickstart for operators (Docker, Umbrel, manual)
  - API reference (all endpoints with request/response examples)
  - Configuration reference (all env vars)
- Add `OPERATOR_GUIDE.md` — step-by-step guide for running a Stable Channels LSP

---

## Key Files to Modify

| File | Changes |
|------|---------|
| `src/bin/lsp_backend.rs` | Health/metrics endpoints, auth middleware, rate limiting, SSE, dashboard API routes, static file serving |
| `src/stable.rs` | Payment retry logic, concurrent safety, force-close reconciliation |
| `src/price_feeds.rs` | Parallel fetching, median with quorum, staleness circuit breaker |
| `src/constants.rs` | Extract to `config.rs` with env var support |
| `src/db.rs` | New queries for dashboard (PnL, history, alerts) |
| `src/audit.rs` | Migrate to `tracing`, structured events |
| `src/lib.rs` | Export new `config` module |
| `Cargo.toml` | Add: `tracing`, `tracing-subscriber`, `prometheus`, `tower-http`, `dashmap` |
| `tests/regtest.rs` | Multi-channel test, API tests |
| `static/` (new) | `index.html`, `dashboard.js`, `dashboard.css` |
| `Dockerfile` (new) | Multi-stage Rust build |
| `docker-compose.yml` (new) | Full stack orchestration |
| `umbrel-app/` (new) | Umbrel manifest + compose + exports |

## Dependencies to Add (Cargo.toml)

- `tracing` + `tracing-subscriber` (structured logging)
- `prometheus` (metrics)
- `tower-http` (static file serving, CORS)
- `dashmap` (concurrent channel map)
- `tower` (rate limiting layer)
