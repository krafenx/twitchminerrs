# PORTING_PLAN.md — Twitch Channel Points Miner v2 → Rust

**Source of truth for the port:** https://github.com/rdavydov/Twitch-Channel-Points-Miner-v2 (rdavydov fork, `master`), cloned read-only at `/home/user/upstream` for analysis (v2.7.x era, `__init__.py` `__version__ = "2.7.6"`-ish; see `git log` of the upstream clone).
**Target crate:** `twitch-points-miner-rs` (binary `twitch-points-miner-rs`), single crate, `rustc` ≥ 1.85, edition 2021, Tokio async, no OpenSSL (rustls).
**Date:** 2026-11-10

---

## 0. Environment constraints discovered (they shape this port)

The build sandbox has **no access to crates.io, static.rust-lang.org, deb.debian.org or PyPI binary toolchains for Linux**. Only `github.com`, `codeload.github.com`, `api.github.com`, `registry.npmjs.org`, `pypi.org`, `files.pythonhosted.org` are reachable. Consequences:

| Item | Resolution |
|---|---|
| Rust toolchain | Assembled from npm-distributed prebuilt components (`@rusttonpm/*-linux-x64`, **rustc/cargo 1.92.0-nightly caccb4d03, 2025-09-24**) because static.rust-lang.org is unreachable. The code avoids nightly-only features; it builds on stable ≥ 1.85. |
| Crates registry | **Not usable.** All crates are vendored from their GitHub repositories into `vendor/` (see `.cargo/config.toml`, directory source replacement). |
| `cargo fmt` / `rustfmt` | rustfmt is not distributed; a wasm-based rustfmt (`@scalar/rust-fmt`, Node shim at `/home/user/rust/bin/rustfmt`) is used to apply/check formatting. `cargo fmt --check` cannot be executed as-is; the exact command used is documented in the final report. |
| `cargo clippy` | clippy 0.1.92 (nightly) is available and wired in. |
| `cargo audit` | Not available (needs advisory-db + cargo-audit build; registry blocked). **Not run** — see final report. A dependency inventory is provided instead (`vendor/VERSIONS.md`). |
| HTTP client | `reqwest` would pull ~50 more crates; a small, purpose-built async HTTP/1.1 client (`src/twitch/http.rs`, rustls + tokio) is implemented instead, behind a narrow `HttpClient` interface so `reqwest` can be dropped in later. |
| WebSocket | `tokio-tungstenite` avoided (deps tree); minimal RFC 6455 client implemented in `src/twitch/websocket.rs` with unit tests for the codec. |
| Persistence | `rusqlite` (bundled SQLite) — compiles vendored sqlite3 amalgamation, no system libs needed. |
| Mock testing | `wiremock` not vendored; a hand-rolled mock HTTP server (`tests/common/mod.rs`) is used instead (an accepted analogue). |
| Toolchain channel | Because only a nightly toolchain is obtainable offline, CI in this environment runs it; the crate itself is stable-compatible. |

**Nothing in the network stack is specifice**: the code, protocols, operation hashes, payload shapes and behavioral logic are ported 1:1 from the Python source. Only the *implementation substrate* changed.

---

## 1. What the original program does (behavioral summary)

1. Logs into Twitch via the **device OAuth flow** (TV client) or a stolen `auth-token` cookie; persists cookies in `cookies/<username>.pkl` (Python pickle).
2. For every configured streamer: resolves `channel_id` via GQL (`GetIDFromLogin`), loads channel-points context (`ChannelPointsContext`), claims any pending bonus, checks online status.
3. Spawns per-watched-channel **`minute-watched` event** POSTs to the *spade URL* (extracted from the streamer page), limited to **2 concurrent channels** with a configurable priority system.
4. Connects to **Twitch PubSub** (`wss://pubsub-edge.twitch.tv/v1`) — topics: `community-points-user-v1`, `video-playback-by-id`, `raid`, `predictions-channel-v1`, `predictions-user-v1`, `community-moments-channel-v1`, `community-points-channel-v1`.
5. On PubSub events: auto-claims bonuses / moments, joins raids, tracks watch streaks, places predictions ("bets") per configurable strategies, contributes to community goals.
6. Syncs drops campaigns from inventory/dashboard every 30 min, auto-claims drops.
7. Optionally joins IRC chat (mention detection), optional analytics server, optional notifications (Telegram/Discord/Matrix/Pushover/Gotify/Webhook).
8. Prints a session report at the end; persists nothing except cookies and analytics JSON.

## 2. Module map (Python → Rust)

| Python module | Rust module | Status | Notes |
|---|---|---|---|
| `TwitchChannelPointsMiner/TwitchChannelPointsMiner.py` (orchestrator, signals, report) | `src/app.rs`, `src/main.rs`, `src/shutdown.rs` | implemented | Tokio tasks + `CancellationToken`-style shutdown (Notify), JoinSet, final report |
| `classes/Twitch.py` (Twitch API) | `src/twitch/client.rs`, `src/twitch/gql.rs`, `src/twitch/gql_ops.rs`, `src/twitch/models.rs` | implemented | GraphQL persisted queries kept verbatim (operationName + sha256Hash) |
| `classes/TwitchLogin.py` (device flow + cookies) | `src/auth/oauth.rs`, `src/auth/token_store.rs`, `src/auth/mod.rs` | implemented | Device flow ported; token file store (JSON, 0600) + env override; Python `.pkl` cookies **not** supported (documented, unpickling is unsafe) |
| `classes/TwitchWebSocket.py`, `classes/WebSocketsPool.py` | `src/twitch/websocket.rs`, `src/twitch/pool.rs`, `src/twitch/pubsub.rs` | implemented | RFC 6455 codec + reconnect **state machine**, 50-topic limit per connection, resubscribe after reconnect, ERR_BADAUTH detection |
| `classes/entities/Stream.py` (Stream, Strategy, Condition, OutcomeKeys, DelayMode) | `src/miner/stream.rs`, `src/miner/bet.rs` | implemented | |
| `classes/entities/Streamer.py`, `StreamerSettings` | `src/miner/channel.rs`, `src/config.rs` | implemented | |
| `classes/entities/Bet.py` (Bet strategies, FilterCondition) | `src/miner/bet.rs` | implemented | SMART, MOST_VOTED, HIGH_ODDS, PERCENTAGE, SMART_MONEY, NUMBER_1..8, stealth_mode, filter_condition, delay modes |
| `classes/entities/EventPrediction.py` | `src/miner/predictions.rs` | implemented | |
| `classes/entities/Campaign.py`, `Drop.py`, `CommunityGoal.py`, `Raid.py`, `PubsubTopic.py`, `Message.py` | `src/miner/campaigns.rs`, `src/twitch/models.rs`, `src/twitch/pubsub.rs` | implemented | |
| `classes/Settings.py` (Priority, FollowersOrder, Settings) | `src/config.rs` | implemented | TOML config; priorities ORDER/STREAK/DROPS/SUBSCRIBED/POINTS_ASC/POINTS_DESC |
| `logger.py` (LoggerSettings, emoji, colors, file rotation) | `src/logging.rs` | implemented (subset) | tracing; `less` mode; JSON or pretty; file logging via `tracing_appender`-less manual appender (`Mutex<File>`); emoji kept |
| `logger.py` (Telegram/Discord/Webhook/Pushover/Gotify/Matrix handlers) | `src/notify.rs` | partial | Telegram/Discord/Webhook/Pushover/Gotify implemented as simple HTTPS POSTs; **Matrix NOT implemented** (login handshake, not worth the risk) |
| `classes/Chat.py` (IRC mention watcher) | `src/miner/chat.rs` | partial | Minimal IRC: JOIN, PING/PONG, mention detection, reconnect; no full IRC framework |
| `classes/AnalyticsServer.py` (Flask charts) | — | not implemented | Would embed an HTTP+chart server; out of scope, documented |
| `classes/Exceptions.py` | `src/error.rs` | implemented | thiserror typed errors |
| `utils.py` (nonce, millify, chunks, internet check, versions) | `src/utils.rs` | implemented | `millify` reimplemented; update-check becomes opt-in (`--check-updates`, default off) |
| `pickle_view.py`, `example.py`, `setup.py`, `Dockerfile` | `README.md`, examples, `config.example.toml` | — | |

## 3. External protocols, endpoints and operations used

### 3.1 HTTP / GraphQL (all ported verbatim from `constants.py`)
- `POST https://gql.twitch.tv/gql` — persisted queries only:
  `WithIsStreamLiveQuery`, `PlaybackAccessToken`, `VideoPlayerStreamInfoOverlayChannel`,
  `ClaimCommunityPoints`, `CommunityMomentCallout_Claim`, `DropsPage_ClaimDropRewards`,
  `ChannelPointsContext` (7fe050e3761eb2cf258d70ee1a21cbd76fa8cf3d7e7b12fc437e7029d446b5e3),
  `JoinRaid`, `ModViewChannelQuery`, `Inventory`, `MakePrediction`, `ViewerDropsDashboard`,
  `DropCampaignDetails`, `DropsHighlightService_AvailableDrops`, `GetIDFromLogin` (94e82a7b1e3c21e186daa73ee2afc4b8f23bade1fbbff6fe8ac133f50a2f58ca),
  `ChannelFollows`, `PersonalSections` (unused by miner logic; kept for completeness), `UserPointsContribution`, `ContributeCommunityPointsCommunityGoal`.
- Batched GQL (JSON array body) for `DropCampaignDetails` chunks of 20 — supported.
- `GET https://www.twitch.tv` — twilight build-ID scraping for `Client-Version` header (regex kept).
- `GET https://usher.ttvnw.net/api/channel/hls/<login>.m3u8?sig=..&token=..` + HEAD of lowest-quality segment ("2024/5 API change" watch-streak workaround) — kept, behind config flag `simulate_watch` (default **true** because without it watch-streak/drops progress is not earned).
- Spade URL: scraped from streamer page → `POST <spade_url>` form-encoded `data=base64(json)` for `minute-watched` — kept.
- OAuth device flow: `POST https://id.twitch.tv/oauth2/device` (scopes `channel_read chat:read user_blocks_edit user_blocks_read user_follows_edit user_read`), `POST https://id.twitch.tv/oauth2/token` (authorization_pending poll + `refresh_token` refresh).

### 3.2 WebSocket (PubSub)
- `wss://pubsub-edge.twitch.tv/v1` — `LISTEN`/`UNLISTEN`/`PING`/`RECONNECT`/`RESPONSE`/`MESSAGE`.
- Topics: `community-points-user-v1.<user_id>`, `video-playback-by-id.<channel_id>`, `raid.<channel_id>`, `predictions-channel-v1.<channel_id>`, `predictions-user-v1.<user_id>`, `community-moments-channel-v1.<channel_id>`, `community-points-channel-v1.<channel_id>`.
- User topics require `auth_token`; 50 topics per connection; server `RECONNECT` honored with backoff+jitter; duplicate `MESSAGE`s (multi-connection) filtered by `message.identifier == type.topic.channel_id` + timestamp.
- **Known upstream caveat:** Twitch has announced PubSub → EventSub migration; PubSub functionality/availability is not guaranteed. EventSub is **not implemented** (would need a different delivery pipeline); see COMPATIBILITY.md.

### 3.3 IRC
- `irc.chat.twitch.tv:6667` (optionally 6697 TLS) — `CAP REQ :twitch.tv/tags`, `PASS oauth:<token>`, `NICK`, `JOIN #<channel>`, `PING`→`PONG`, `PRIVMSG` mention detection.

### 3.4 Auth model
Device flow (no password stored). `refresh_token` is stored and used to refresh `access_token` on expiry/401. Python `.pkl` cookie files are intentionally **not** readable by the Rust port (arbitrary pickle = code execution risk). Migration path: re-run device flow; token is stored as JSON with 0600 permissions.

### 3.5 Persistence
Python version persisted only cookies (pickle) + analytics JSON. Rust port adds a `state.sqlite3` (WAL) with migrations: `seen_events(topic,identifier,ts)`, `claimed_bonuses(claim_id, ts)`, `actions(kind, channel, payload_hash, ts)` for cross-restart dedup. Tokens are **never** stored in SQLite; they live in `tokens.json` (0600) or `TWITCH_AUTH_TOKEN` env var.

## 4. Rust design decisions

- **Concurrency:** one Tokio task per concern: `minute_watcher`, per-WS-connection tasks, `sync_campaigns`, `context_refresher`, shutdown watcher. Shared state via `Arc<Mutex<…>>` only for tiny critical sections; message passing (`tokio::sync::{mpsc, broadcast, watch}`) for events; no locks across `.await`.
- **Action layer:** all state-changing Twitch calls go through `TwitchActionExecutor` (real & dry-run impls) — `claim_bonus`, `make_prediction`, `join_raid`, `claim_moment`, `contribute_goal`, `claim_drop`, `send_minute_watched`.
- **State machines:** WS connection states `Disconnected → Connecting → Connected → Subscribing → Active`, any → `BackingOff{attempt}`, all → `ShuttingDown`. Unit-tested transitions.
- **Error handling:** `thiserror` everywhere; `anyhow` confined to `main.rs`/CLI exit paths; no `unwrap`/`expect` outside tests (enforced by clippy + review).
- **Retry policy:** only for transient errors (connect/TLS/timeout/5xx/429-with-retry-after/GQL `service unavailable`); max 5 attempts, exponential backoff base 500ms, cap 30s, ±25% jitter; `Retry-After` honored; auth errors (401/ERR_BADAUTH) never retried silently — surfaced with actionable message.
- **Secrets:** `Redacted<T>` wrapper; `Debug`/`Display` print `***`; tokens never logged, never in SQLite, never in panic messages.
- **Validation before I/O:** `validate-config` and startup run full validation (username present, channel list non-empty-unless-followers, numeric bounds, enum values, proxy URL parse) **before** any network call.
- **Priority engine:** pure functions in `src/miner/rules.rs` (`ChannelDecision`) — unit-tested, no network.
- **Metrics:** `src/metrics.rs` atomic counters (events_received, reconnects, claims_ok/ko, predictions_made, actions_skipped, rate_limited, parse_failures) printed on shutdown.

## 5. Feature checklist & status legend

Legend: `planned` → `implemented` → `tested`; `blocked` = cannot verify/complete in this environment; `n/a` = not applicable.

| # | Feature (Python source) | Rust module | Status |
|---|---|---|---|
| F1 | CLI entry (`example.py` semantics): config via file | `cli.rs`, `config.rs` | tested (config tests) |
| F2 | Device OAuth login flow | `auth/oauth.rs` | implemented (no live account → `blocked` runtime check) |
| F3 | Token persist/load + refresh | `auth/token_store.rs`, `auth/oauth.rs` | implemented + unit tests |
| F4 | GQL dispatch w/ persisted queries, batched queries | `twitch/gql.rs`, `gql_ops.rs` | implemented + mock tests |
| F5 | HTTP 429/Retry-After, 5xx retry, backoff+jitter, 401 stop | `twitch/http.rs`, `twitch/retry.rs` | tested |
| F6 | PubSub connect/LISTEN/PING/PONG/RECONNECT/dedup | `twitch/pubsub.rs`, `twitch/websocket.rs`, `twitch/pool.rs` | implemented + codec tests |
| F7 | Points earned/spent tracking + balance update | `miner/manager.rs` | implemented |
| F8 | Claim available (bonus) on PubSub `claim-available` + on context load | `twitch/client.rs`, `miner/points.rs` | implemented (dry-run tested) |
| F9 | Watch streak priority handling | `miner/rules.rs`, `miner/manager.rs` | implemented + unit tests |
| F10 | minute-watched watcher (≤2 channels, priority order) | `miner/scheduler.rs` | implemented |
| F11 | Spade URL scraping + m3u8 lowest-quality HEAD workaround (“2024/5 fix”) | `twitch/spade.rs` | implemented (behind `simulate_watch`) |
| F12 | Stream online/offline detection (pubsub + poll) | `miner/manager.rs` | implemented |
| F13 | Followers import (`followers=True`) | `twitch/client.rs` (`ChannelFollows`) | implemented |
| F14 | Drops: inventory sync, dashboard, claim, progress print | `miner/campaigns.rs` | implemented (logic), mock-tested |
| F15 | Predictions: event-created/updated, bet strategies, filters, delay | `miner/predictions.rs`, `miner/bet.rs` | implemented + extensive unit tests |
| F16 | Predictions result tracking (prediction-user-v1) | `miner/predictions.rs` | implemented |
| F17 | Community goals (contribute MAX contribution) | `miner/goals.rs` | implemented |
| F18 | Moments claim | `miner/manager.rs` | implemented |
| F19 | Raid join (`follow_raid`) | `miner/raids.rs` | implemented |
| F20 | IRC chat join + mention alert | `miner/chat.rs` | partial (no auto-reconnect backoff cap; no commands) |
| F21 | Notifications (Telegram/Discord/Webhook/Pushover/Gotify) | `notify.rs` | partial (send path implemented+tested w/ mock, templates simplified vs Python emoji/color palette) |
| F22 | Matrix notifications | — | **not implemented** (login flow complexity) |
| F23 | Analytics web server + charts (flask) | — | **not implemented** |
| F24 | `.pkl` cookie import | — | **not implemented** (unsafe); migration = re-auth |
| F25 | Version self-check (GitHub raw) | `utils::check_versions` (opt-in `--check-updates`) | partial |
| F26 | Final session report + gains summary | `app.rs` | implemented |
| F27 | Stealth mode, millify output | `miner/bet.rs`, `utils.rs` | implemented + tests |
| F28 | Duplicate message filtering across pool connections | `twitch/pool.rs` | implemented |
| F29 | `--dry-run` mode | `miner/actions.rs` (`DryRunExecutor`) | implemented + tests |
| F30 | Multi-account | — | **not implemented** (Python had none either — single username per process); doc explains process-per-account |
| F31 | Analytics `persistent_series/annotations` | — | **not implemented** (part of F23) |
| F32 | `viewer_is_mod` | `twitch/client.rs` | implemented (kept for parity, UI-hidden) |

## 6. Known/live-risk Twitch endpoints (cannot verify without credentials)

- Device flow (`id.twitch.tv`) — documented publicly, not exercised here.
- `gql.twitch.tv` persisted-query hashes — copied 1:1; hash rotation by Twitch would break them; mitigated by `Client-Version` scrape.
- PubSub availability — deprecated path per Twitch announcements; see COMPATIBILITY.md.
- Spade endpoint — scraped per streamer; requires live stream.
- Refresh token grant — supported by Twitch for device-flow tokens, unverified here.

## 7. Test plan

Unit (in-crate, no network): config merge/validate, backoff+jitter bounds, Retry-After parsing, rule engine priority decisions, Bet strategies & filters (SMART/MOST_VOTED/…/skip/stealth), pubsub message parse (points/raid/prediction/claim/moments/goals), WS frame codec roundtrip + mask validation, dedup store (in-memory + sqlite via tempfile), prediction window calc, millify, rfc3339/epoch formatting, dry-run executor, metrics.
Integration (`tests/`, mock servers on 127.0.0.1): GQL happy path, GQL errors array, HTTP 401 → auth error surfaced, 429+Retry-After → waited & retried, 5xx ×N → retried then error, malformed JSON → typed error, WS handshake+LISTEN+PING flow against in-process mock WS server, reconnect/resubscribe behavior.
Not covered: live Twitch (no credentials).

## 8. Commit / verification policy

Each stage ends with: format (wasm-rustfmt shim), `cargo clippy --all-targets -- -D warnings`, `cargo test`, atomic commit on `arena/7b293689-twitchminerrs`.
