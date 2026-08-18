# Omnigent Server Subsystem — Research Report

> Produced 2026-08-18 by a read-only Claude research subagent (one of four parallel passes
> over this checkout) that fed the "Omnigent Anatomy" architecture walkthrough.
> All file:line citations refer to the working tree at `apps/omnigent` on that date.
> Uncertainties are listed at the end rather than guessed at.

## Module inventory

| Path | Responsibility |
|---|---|
| `omnigent/cli.py:2916` (`omni server`/`omnigent server` group) | Real process entrypoint; builds app via `create_app()` and runs uvicorn |
| `omnigent/__main__.py:3` | `python -m omnigent` → same CLI |
| `omnigent/host/local_server.py` (906 lines) | **Not a second server.** Pidfile-tracked supervisor that spawns `sys.executable -m omnigent.cli server ...` as a subprocess on `127.0.0.1` (default port 6767) so repeated local CLI invocations reuse one instance. The spawned process is the *identical* FastAPI server described below — locally, host/runner/CLI/browser all still talk to it over the same WS tunnel protocols as a remote deployment |
| `omnigent/server/app.py` | FastAPI app factory `create_app()` (`app.py:1063`), lifespan (`app.py:1231`), router mounting, middleware |
| `omnigent/server/routes/*.py` | REST/WS route handlers (table below) |
| `omnigent/server/auth.py`, `oidc.py`, `oidc_access.py`, `accounts_*.py`, `passwords.py` | Auth (see Auth section) |
| `omnigent/server/host_registry.py`, `presence.py`, `mcp_pool.py`, `smart_routing.py`, `managed_hosts.py`, `performance_metrics.py`, `admin_list.py`, `bundles.py` | In-memory registries, presence tracking, MCP proxy pool, model routing, admin roster, metrics |
| `omnigent/host/connect.py`, `identity.py`, `frames.py`, `daemon_launch.py` | Host daemon: dials out to server's `/v1/hosts/{id}/tunnel`, spawns runner subprocesses |
| `omnigent/runner/transports/ws_tunnel/*` | Runner-side WS tunnel client (`serve.py`, `frames.py`, `registry.py`, `transport.py`) |
| `omnigent/db/db_models.py`, `migrations/` | SQLAlchemy models + Alembic migrations |
| `omnigent/stores/*` | Store layer (repository pattern) over the DB models |
| `omnigent/entities/*` | Plain dataclasses/Pydantic domain objects (`Conversation`, `Agent`, `Permission`, `Policy`, etc.) |

**Framework**: FastAPI on Starlette (`server/app.py:15-22`), app = `FastAPI(title="Omnigent Server", lifespan=_lifespan)` at `app.py:1383`. No `CORSMiddleware` found anywhere (grepped) — likely same-origin via the bundled SPA static mount, or CORS handled by a deployment-side proxy; unconfirmed. Middleware present: WS-metrics counter, `WebSocketOriginMiddleware` (CSWSH guard), request-metrics, gzip on the static SPA mount.

## Route inventory (selected)

`sessions.py` is the dominant file (~21.8k lines, ~60 endpoints under `/v1/sessions`). Highlights:

| Path | Method(s) | file:line | Purpose |
|---|---|---|---|
| `/v1/sessions` | POST, GET | `sessions.py:14287,14856` | create/list sessions |
| `/v1/sessions/{id}` | GET/PATCH/DELETE | `sessions.py:14744,15429,20435` | session CRUD (incl. `include_items` snapshot) |
| `/v1/sessions/updates` | **WebSocket** | `sessions.py:15164` | live session-list push, replaces 4s HTTP poll |
| `/v1/sessions/{id}/stream` | GET (**SSE**, `text/event-stream`) | `sessions.py:20245` | live-tail of turn events for one session |
| `/v1/sessions/{id}/events` | POST | `sessions.py:19157` | submit input/interrupt/approval into a session |
| `/v1/sessions/{id}/resources[...]` | GET/POST/PUT/PATCH/DELETE | `sessions.py:17399-18930` | env filesystem, terminals, files, browser actions |
| `/v1/hosts`, `/v1/hosts/{id}` | GET | `hosts.py:321,371` | list/get registered hosts |
| `/v1/hosts/{id}/runners` | POST | `hosts.py:408` | launch a runner on a connected host |
| `/v1/hosts/{id}/tunnel` | **WebSocket** | `host_tunnel.py:121` | host daemon control-plane tunnel |
| `/v1/runners/{id}/tunnel` | **WebSocket** | `runner_tunnel.py:344` | runner HTTP-over-WS tunnel |
| `/v1/runners/{id}/token` | POST | `runner_tunnel.py:284` | mint runner binding token |
| `/v1/sessions/{id}/resources/terminals/{tid}/attach` | **WebSocket** | `terminal_attach.py:131` | browser xterm.js ↔ runner PTY bridge |
| `/auth/*` (OIDC) | GET/POST | `routes/auth.py:131-558` | login/callback/invite/logout/CLI device-code |
| `/auth/*` (accounts) | GET/POST/DELETE | `routes/accounts_auth.py:244-739` | password login/register/setup/magic-link/user admin |

Every non-SSE, non-WS content endpoint is plain REST/JSON validated by ~61 Pydantic models in `server/schemas.py` (4066 lines).

## Data model summary

Two SQLAlchemy `DeclarativeBase`s, `OmnigentBase` and `ConversationBase` (`db_models.py:38-58`), can point at separate physical DBs. Every table's PK leads with `workspace_id` (tenant partition, added by migration `r1a2b3c4d5e6`).

| Table | Key fields | Durable vs derived | file:line |
|---|---|---|---|
| `agents` | `bundle_location` (artifact-store key, not blob), `kind`, `description` (compressed) | Durable metadata; blob external | `db_models.py:112-169` |
| `files` | `filename`, `bytes`, `content_type`, `session_id` | Durable metadata; bytes live in `ArtifactStore` (local/S3/Databricks volumes) | `db_models.py:172-214` |
| `users` | `is_admin`, `password_hash`, `last_login_at` | Durable | `db_models.py:217-256` |
| `account_tokens` | `kind` (invite/magic), `user_id`, `redeemed_at`, `expires_at` | Durable, short-TTL single-use | `db_models.py:259-317` |
| `session_permissions` | `(user_id, conversation_id)` → `level` (1=read/2=edit/3=manage) | Durable ACL | `db_models.py:320-371` |
| `omnigent_conversation_metadata` | `runner_id`, `host_id`, `session_state`/`session_usage` (compressed JSON) | Persisted-as-source-of-truth runtime snapshot | `db_models.py:374-422` |
| `agent_configuration` | `agent_id`, `model_override`, `harness_override` | Durable | `db_models.py:425-480` |
| `conversations` | `title`, `parent_conversation_id`, `root_conversation_id`, `next_position` (monotonic allocator) | Durable | `db_models.py:483-561` |
| `conversation_items` | `position` (unique per conversation), `data` (JSON), `type`/`status` | Durable, **append-only** — no update path found for item bodies | `db_models.py:564-647` |
| `conversation_items_fts` | FTS5 virtual table (SQLite/Cloudflare D1 only) | Fully derived index | `db/utils.py:708-748` |
| `comments`, `policies`, `hosts`, `user_daily_cost` | — | `hosts.status`/`updated_at` doubles as heartbeat (semi-derived); `user_daily_cost` is an explicit O(1) rollup cache | `db_models.py:705-1051` |

**Engines**: SQLite (default, `sqlite:///omnigent.db`, WAL+PRAGMAs, `db/utils.py:231-286`), PostgreSQL via `psycopg` (`db/utils.py:179-286`), Databricks Lakebase (Postgres + OAuth token refresh, `db/utils.py:35-173`), MySQL via `pymysql`, and Cloudflare D1 (SQLite-over-HTTP). All access is synchronous SQLAlchemy run via `asyncio.to_thread` — no async DB drivers.

**Compression**: `CompressedText` type transparently zstd-compresses 6 "opaque" columns never queried in SQL (`agents.description`, `session_state`/`session_usage`/`terminal_launch_args`, `comments.body`/`anchor_content`) — `db/compression.py:1-59`.

**Notable migration**: `p1a2b3c4d5e6_remove_all_fks.py` drops all remaining DB-enforced foreign keys per internal "Rule R032" — the application layer is now solely responsible for cascading deletes (confirmed by hand-written cleanup in `host_store.py:736-757`).

**Blob storage**: `ArtifactStore` (local/S3/Databricks-volumes, uniform `put/get/delete/exists` interface) holds all file/bundle bytes; the SQL DB stores only pointer keys — confirmed no blob columns exist for `agents.bundle_location` or `files`.

## Session/conversation model & client delivery

"Session" (API/product term) and "Conversation" (`entities/conversation.py:26`, DB `conversations` table) are the same entity. Key fields: identity/hierarchy (`root_conversation_id`, `parent_conversation_id`, `kind`), `agent_id`, `runner_id`/`host_id` placement, `workspace` (immutable realpath), `session_state`/`session_usage` (mutable JSON), and per-session overrides (`model_override`, `harness_override`, etc.) — `entities/conversation.py:191-216`.

Messages/tool-calls/reasoning are stored as **append-only `conversation_items` rows ordered by a monotonic `position`** allocated under a row lock (`sqlalchemy_store.py:1770-1849`), not simple timestamps. `designs/OBSERVABILITY.md:424` explicitly notes a lower-level wire-format "durable event log" is a **future, not-yet-built** Phase-3 item — `conversation_items` is today's closest equivalent, already durable.

**Protocol summary**:

| Mechanism | Direction | Purpose | file:line |
|---|---|---|---|
| `GET /v1/sessions/{id}/stream` (SSE) | server→client | live turn-event tail (no replay; client re-fetches snapshot on reconnect) | `sessions.py:20245` |
| `WS /v1/sessions/updates` | bidirectional | session-list live diff push | `sessions.py:15164` |
| In-process pub/sub `session_stream.publish/subscribe` | server-internal | fans SSE out from workflow threads; per-conversation `asyncio.Queue`, no DB, **events dropped if no subscriber** (at-most-once) | `runtime/session_stream.py:1-108` |
| `WS /v1/hosts/{id}/tunnel` | bidirectional | host control-plane (launch/stop runner, fs ops) | `host_tunnel.py:121` |
| `WS /v1/runners/{id}/tunnel` | bidirectional | runner HTTP-over-WS + terminal-channel multiplex | `runner_tunnel.py:344` |
| `WS /v1/sessions/{id}/resources/terminals/{tid}/attach` | bidirectional | raw PTY bytes, proxied through the runner tunnel if remote | `terminal_attach.py:131` |
| `presence.py` in-memory registry | server-internal | who's viewing a session tree; 15s reconnect grace window; dies with process, never persisted | `presence.py:1-65` |

Clients (web/mobile/desktop, and the `sdks/python-client/omnigent_client` package) are **thin projections**: the SDK's `BlockStream` reconstructs semantic UI blocks purely from the SSE event sequence, with no local persistent store found in either SDK package — consistent with `designs/CUJ-ANALYSIS.md:457` explicitly labeling "conversation history (source of truth)" as server-side.

## Auth

Three mutually-exclusive user-identity modes, selected by `OMNIGENT_AUTH_PROVIDER` (`server/auth.py:266-307`):
- **`header`** (default) — trusts an upstream-injected identity header (`X-Forwarded-Email` by default); this is the mechanism Databricks Apps SSO uses (`docs/databricks.md:53-55`, confirmed via `server/auth.py:617-619`) — **there is no Databricks-specific OIDC code path**, Databricks flows through generic trusted-header auth.
- **`oidc`** — full authorization-code + PKCE flow, generic-OIDC-discovery or hardcoded GitHub provider (`server/oidc.py:26-355`).
- **`accounts`** — built-in argon2id password auth, first-user-becomes-admin bootstrap that self-disables once any admin exists (`accounts_bootstrap.py:1-24,243-389`).

Both OIDC and accounts share an HS256 JWT session cookie (`__Host-ap_session`) validated in `UnifiedAuthProvider._check_cookie` (`server/auth.py:477-539`).

There is **no `auth_tokens` table** — the token table added by the migration named for it is actually **`account_tokens`** (`db_models.py:259-317`), storing invite/magic-link tokens *verbatim* (not hashed), single-use via atomic `redeemed_at IS NULL` update.

**Host auth** (`host_tunnel.py:121-235`, before `ws.accept()`): either (a) a user's Bearer JWT (same cookie/session mechanism, for user-owned `omnigent host` daemons), or (b) `X-Omnigent-Host-Token` — a random launch token minted for server-provisioned managed sandboxes, stored server-side only as a SHA-256 digest (`hosts.token_hash`, never the raw token — `host_store.py:155-168`), scoped to one `host_id`, implicitly rotated on sandbox relaunch. Runner sub-processes get a separate one-time `binding_token` minted per-launch, independent of host auth.

## System of record analysis

**Server is durably authoritative for**: full conversation history (`conversations`/`conversation_items`, monotonic-position append log), the host registry's persistent fields (`hosts` table, upserted on connect/heartbeat), policies, permissions, comments, users, and agent configs — all backed by the SQL DB.

**Host** holds only in-memory state locally (live WS connection object, spawned-runner subprocess handles) and *syncs up* to the server via frames on its outbound tunnel; the server never opens a connection to a host, and never mirrors the host's filesystem — it only *pulls* on demand via request/response frames (`host.stat`, `host.list_dir`, etc.).

**Runner** holds the actual agent process memory, local sandbox workspace files, and tmux/PTY state entirely locally; these never get replicated to the server — only the resulting conversation turns (via the runner tunnel's HTTP-over-WS RPC) get persisted as `conversation_items` as a side effect of the server handling each response.

**Sync direction**: overwhelmingly push/RPC-over-persistent-tunnel, always initiated outbound by the host/runner (never server→host/runner connection). The one asynchronous, unprompted push from host to server is `host.runner_exited` (crash notification) — otherwise every frame is a response to a server-initiated request frame sent down the already-open tunnel. `host/local_server.py` does not change this picture: it only supervises a subprocess running the identical server code, so even fully-local single-machine setups route host/runner/client traffic through the same `/v1/hosts/…/tunnel` and `/v1/runners/…/tunnel` WS protocol against a loopback instance.

## Uncertainties (consolidated across sub-investigations)

- No `CORSMiddleware` registration found anywhere in `server/app.py` — CORS handling location (proxy vs none-needed) is unconfirmed.
- `sessions.py` is ~21.8k lines; route inventory was built from decorator/path extraction, not full manual reads of every of the ~60 handler bodies.
- A `/health/subscribe` WebSocket route is listed in `designs/CUJ-ANALYSIS.md:456` but could not be located by grep in `server/` — may be named differently, or the doc may be stale/aspirational.
- The Phase-3 "durable event log" in `designs/OBSERVABILITY.md:424` is a proposed future addition; unclear if any partial implementation already exists outside what was searched.
- The exact call site that mints a managed-sandbox host's initial launch token was not located within the files reviewed.
- `omnigent/runner/transports/ws_tunnel/registry.py` (978 lines) and `serve.py` (948 lines) — runner-side reassembly/backpressure and reconnect logic were not read in full; assumed symmetric to the host side based on shared constants, not independently confirmed.
- Store-layer internals for `conversation_store/sqlalchemy_store.py` (2851+ lines, search/forking/daily-cost logic) and the three `artifact_store` backends were reviewed only at the method-signature level, not fully line-by-line.
