# Omnigent HOST + RUNNER — Research Report

> Produced 2026-08-18 by a read-only Claude research subagent (one of four parallel passes
> over this checkout) that fed the "Omnigent Anatomy" architecture walkthrough.
> All file:line citations refer to the working tree at `apps/omnigent` on that date.
> Uncertainties are listed at the end rather than guessed at.

No `designs/RUNNER.md` exists in this checkout. `designs/` has other docs (CUJ-ANALYSIS, OBSERVABILITY, SANDBOX_CREDENTIAL_PROXY, CLI_CONTRACT, harness-plugin-interface, etc.) but nothing host/runner-specific. Docstrings in `git_worktree.py:6` and `frames.py:237,360,405` cite `designs/SESSION_GIT_WORKTREE.md` / `designs/SESSION_WORKSPACE_SELECTION.md`, but neither exists in this checkout — likely relocated or removed. The protocol is documented only inline in `frames.py`/`connect.py` docstrings.

## 1. Module Inventory

**Host** (`omnigent/host/`):
| Path | Size | Responsibility |
|---|---|---|
| `connect.py` | 90KB/2014L | `HostProcess` — main WS loop, reconnect/backoff, spawns+watches runner subprocesses, worktree ops, orphan reaper |
| `frames.py` | 45KB/1263L | 18-kind `HostFrameKind` enum + 20 frame dataclasses, encode/decode |
| `identity.py` | 3.9KB/108L | `HostIdentity`, load/create from `~/.omnigent/config.yaml` or managed-sandbox env vars |
| `daemon_launch.py` | 10.6KB/251L | CLI-side helpers hitting server's `/v1/hosts/{id}/runners` REST endpoint (launch-or-reuse, retries) |
| `_daemon_entry.py` | 2.1KB/68L | `python -m omnigent.host._daemon_entry` process entry point |
| `local_server.py` | 38KB/906L | Spawns/manages a full local `omnigent server` subprocess on loopback for local mode |
| `git_worktree.py` | 18.7KB/463L | `git` subprocess wrapper: validate/create/list/remove worktrees |

**Runner** (`omnigent/runner/`):
| Path | Size | Responsibility |
|---|---|---|
| `app.py` | 883KB/19178L | FastAPI app (`create_runner_app`, `app.py:7880`) — session/terminal/filesystem/mcp routes, per-harness auto-terminal-create |
| `_entry.py` | 53KB/1244L | Runner subprocess entry: env wiring, spec resolver, boots WS tunnel, `main()` |
| `tool_dispatch.py` | 260KB/6403L | `execute_tool`/`dispatch_tool_locally` — central tool-call executor, sub-agent spawn |
| `mcp_manager.py` | 32KB/810L | `RunnerMcpManager` — direct stdio MCP pool, no-AP/test mode |
| `proxy_mcp_manager.py` | 14KB/327L | `ProxyMcpManager` — proxies MCP calls through server, AP mode |
| `pending_approvals.py` | 9KB/207L | Global `asyncio.Future` registry awaiting ASK verdicts |
| `policy.py` | 13KB/336L | `RunnerToolPolicyGate` — local fast-path policy enforcement |
| `resource_registry.py` | 57KB/1372L | `SessionResourceRegistry` — terminal/environment lifecycle |
| `environment_filesystem.py` | 32KB/844L | `CallerProcessFilesystem` — typed FS adapter over sandboxed `OSEnvironment` |
| `routing.py` | 10KB/280L | `RunnerRouter` — **server-side** conversation→runner dispatch code that lives in this package (imports `ConversationStore`/`TunnelRegistry`) |
| `identity.py` | 6KB/142L | Runner identity, tunnel binding-token constants, secret-stripping |
| `uc_function.py` | 9.5KB/244L | Executes Databricks Unity Catalog SQL functions |

**Transports** (`omnigent/runner/transports/`):
`__init__.py` documents 3 phases: UDS (`uds.py`, legacy/test), TCP (`tcp.py`, legacy/test), WS tunnel (`ws_tunnel/`, production). `ws_tunnel/frames.py` (wire dataclasses), `limits.py` (timing/size constants), `registry.py` (server-side `runner_id→WS` map), `serve.py` (runner-side WS client loop), `transport.py` (`WSTunnelTransport(httpx.AsyncBaseTransport)`, server-side).

**Sandboxing**: `cli_sandbox.py` is a false lead — it's `omnigent sandbox {create,auth,connect}`, CLI for provisioning a *remote cloud sandbox* (Modal/Daytona/Lakebox/Islo) to host an Omnigent **host** process, unrelated to OS process sandboxing. Actual sandboxing lives in `inner/sandbox.py` (policy/registry), `inner/bwrap_sandbox.py` (Linux), `inner/seatbelt_sandbox.py` (macOS), re-exported via `sandbox/bwrap.py`/`sandbox/seatbelt.py`.

## 2. Host Frame Protocol Table (`host/frames.py`)

All frames JSON dataclasses under `HostFrameKind` enum (`frames.py:37`):

| Frame | Direction | Purpose | Key fields |
|---|---|---|---|
| `HostHelloFrame` (`:64`) | host→server | Register host on fresh connect | `version, name, runners[], configured_harnesses` |
| `HostLaunchRunnerFrame` (`:93`) | server→host | Spawn a runner | `request_id, binding_token, workspace, session_id, harness` |
| `HostLaunchRunnerResultFrame` (`:123`) | host→server | Launch outcome | `request_id, status, runner_id, error, error_code` |
| `HostStopRunnerFrame` (`:150`) | server→host | Terminate a runner | `request_id, runner_id` |
| `HostStopRunnerResultFrame` (`:164`) | host→server | Stop outcome | `request_id, status, error` |
| `HostRunnerExitedFrame` (`:180`) | host→server | One-way unexpected-death report | `runner_id, error` |
| `HostStatFrame`/`Result` (`:204/227`) | server→host / reply | Stat a filesystem path | `path` / `exists, type, canonical_path` |
| `HostListDirFrame`/`Result` (`:297/328`) | server→host / reply | Directory browse | `path, limit, after, before` / `entries[], has_more` |
| `HostCreateWorktreeFrame`/`Result` (`:357/377`) | server→host / reply | Create git worktree | `repo_path, branch_name, base_branch` / `worktree_path, branch` |
| `HostRemoveWorktreeFrame`/`Result` (`:401/425`) | server→host / reply | Remove worktree | `worktree_path, branch, delete_branch` |
| `HostListWorktreesFrame`/`Result` (`:441/459`) | server→host / reply | List worktrees | `repo_path` / `worktrees[]` |
| `HostCreateDirFrame`/`Result` (`:479/498`) | server→host / reply | mkdir on host | `path` |

Plus app-level `PingFrame`/`PongFrame` (`ws_tunnel/frames.py:38-39`) for tunnel keepalive on both host and runner tunnels.

## 3. Runner Internal Architecture — Data Flow

Harness subprocess → `app.py`'s `/v1/sessions/{id}/mcp/execute` (`app.py:18151`), or a harness's `policy_evaluation.requested` SSE event intercepted → `_evaluate_policy_via_omnigent` (`app.py:6430`) POSTs to server `/policies/evaluate`, relays verdict back as `policy_verdict`. For runner-owned categories, `tool_dispatch.execute_tool` (`tool_dispatch.py:4389`) either delegates MCP calls to the manager (`mcp_manager.call_tool`) or dispatches `sys_os_*`/REST/terminal/subagent tools locally.

Two enforcement points:
- **Local fast path**: `RunnerToolPolicyGate._evaluate_policies` (`policy.py:228`) — only used with `RunnerMcpManager` (no-AP/test mode); DENY short-circuits, ASK escalates via `pending_approvals.wait_for_user_approval` (`pending_approvals.py:133`), a Future keyed by `elicitation_id`, resolved when the server POSTs to `/v1/sessions/{id}/events`.
- **Server round-trip** (AP mode, `ProxyMcpManager`): `_evaluate_policy_via_omnigent` (`app.py:6430`) — TOOL_CALL fails **closed** on error, LLM_REQUEST/RESPONSE and TOOL_RESULT fail **open** (`app.py:6449-6459`); ASK parks server-side, runner awaits via `pending_approvals` with a 1-day timeout (`app.py:6497-6505`, `pending_approvals.py:43`).

`mcp_manager.py` vs `proxy_mcp_manager.py` (explicit doc at `proxy_mcp_manager.py:14-22`): `RunnerMcpManager` (no-AP/test) owns direct stdio MCP subprocess connections (LRU pool, capacity 8, `mcp_manager.py:44`) and enforces policy locally. `ProxyMcpManager` (AP mode, production) holds no connections — every `call_tool` (`:178`) POSTs JSON-RPC to the server's `/v1/sessions/{id}/mcp`, which owns real connections centrally and enforces policy there. Both expose the same interface, interchangeable at dispatch sites.

Harness spawning: `POST /v1/sessions` (`app.py:8767`) triggers per-harness `_auto_create_{claude,codex,cursor,goose,...}_terminal` functions (e.g. `app.py:5347` for Claude), which call `resource_registry.launch_required_terminal`/`launch_auxiliary_terminal` (`resource_registry.py:758,791`) → `_launch_terminal_with_lifecycle` (`:824`). The actual subprocess/tmux mechanics are **outside `runner/`**, owned by `HarnessProcessManager` (`omnigent.runtime.harnesses.process_manager`, wired in at `_entry.py:865`) — not directly `subprocess.Popen` inside `app.py`.

Agent spec resolution: `_entry.py:849 create_app()` builds a `spec_resolver` (`:952`) wrapping `_resolve_agent_spec_from_server` (`:776`) — fetches `GET /v1/sessions/{id}/agent/contents`, checks `X-Agent-Session-Scoped` header (fail-safe: unknown → no `${VAR}` expansion, `:823-826`), caches the extracted bundle per runner, calls `omnigent.spec.load(...)` → `ResolvedSpec` (`app.py:6622`).

## 4. Transport Architecture — Definitive Answer

**Each runner opens its own dedicated WebSocket directly to the server; it is NOT tunneled through the host's WS connection.** Two independent WS clients:
- Runner→server: `serve.py:547` `websockets.connect(tunnel_url,...)`, `tunnel_url` from `_tunnel_url(server_url, runner_id)` (`serve.py:914`), hitting server route `@router.websocket("/runners/{runner_id}/tunnel")` (`server/routes/runner_tunnel.py:344`).
- Host→server: separate connection in `host/connect.py` against `server/routes/host_tunnel.py`.

The server does multiplex many server-side httpx requests onto one runner's WS via `WSTunnelTransport` (`transport.py:104-113`) and the `TunnelRegistry` (`registry.py:1-9`), but that WS belongs solely to the runner (dialed by the runner itself). The host only launches the subprocess and injects env (`host/connect.py:520-548`) and watches/reports its exit — it does not proxy runner traffic.

`RUNNER_TUNNEL_BINDING_TOKEN_ENV_VAR = "OMNIGENT_RUNNER_TUNNEL_BINDING_TOKEN"` (`runner/identity.py:21`). **Set** at `host/connect.py:547` (per-launch, injected into the runner subprocess env) and `cli.py:2804` (local `omnigent run`, `secrets.token_urlsafe(32)` fallback). **Read** at `runner/_entry.py:580-594`. Sent back as HTTP header `X-Omnigent-Runner-Tunnel-Token` on the WS handshake (`serve.py:546`). Purpose is dual: auth *and* session binding — `token_bound_runner_id(token)` (`identity.py:98-115`) SHA-256-hashes the token into a deterministic `runner_id`; the server route's path param must match, cryptographically preventing one caller from claiming another's runner id on a shared server. Explicitly stripped before any spawned child process inherits it (`RUNNER_AUTH_SECRET_ENV_VARS`, `identity.py:47-55`).

Ping/Pong: library-level keepalive `ping_interval=30s, ping_timeout=90s` (`serve.py:555-556`, `ws_tunnel/limits.py:24-25`) plus app-level `PingFrame`/`PongFrame` JSON frames sent every 30s by the server, echoed by the runner (`serve.py:646-647`); ~3 missed pongs (`limits.py:9-10` comment) declares the peer dead at ~90s.

## 5. Reconnection / Retry / Backoff (host)

Constants (`host/connect.py:248-250`): `_RECONNECT_BASE_S=0.5`, `_RECONNECT_CAP_S=10.0`, `_RECONNECT_JITTER=0.5`. `HostProcess.run()` (`:1612`) loops `_connect_and_serve()` (`:1726`); backoff resets to base on success (`:1635`). Fatal `HostConnectError` (auth/authorization/outdated server) re-raises without retry → `SystemExit(1)` (`:2007-2013`). Other failures: "recycle" signals (close code 1012/1001, "going away", or 502/no-close-frame on non-loopback servers) get an immediate base-delay retry; otherwise exponential backoff: `backoff = min(backoff*2*(1+random()*jitter), cap)` (`:1673-1697`). Loopback servers skip the fast-recycle path deliberately to avoid re-registration flapping (`:1663-1672`). Login-redirect fatal threshold: `_LOGIN_REDIRECT_FATAL_ATTEMPTS=3` (`:480`), but only for hosts that have never connected — an already-connected host retries redirects forever.

## 6. Lifecycle / Failure Narrative

Runner launch: `_handle_launch` (`connect.py:1029`) validates workspace, builds env via `_build_runner_env` (allowlist-based, `:261,499`), `subprocess.Popen([sys.executable, "-m", "omnigent.runner._entry"], ...)` (`:1097-1111`). Each launch spawns `_watch_runner` (`:1192`) polling `proc.poll()` every `_RUNNER_WATCH_INTERVAL_S=0.5s` (`:135`).

Crash path: unexpected exit while still tracked → composes exit code + log tail → `_report_runner_exit` (`:1220-1244`) encodes `HostRunnerExitedFrame` and sends over the host's live WS. If send fails (tunnel down), it's parked in `self._unreported_exits[runner_id]` (`:1244`) and flushed on next successful reconnect (`:1863-1866`). Server-side, `host_tunnel.py:452-464` matches the frame, logs it, calls `runner_exit_reports.record(...)` — surfacing "offline, and here's why" to status pollers rather than a bare timeout. **No automatic respawn/session-resume is triggered by this frame** — it's purely diagnostic. Session resumption instead lives client-side in `daemon_launch.launch_or_reuse_daemon_runner` (`daemon_launch.py:166-229`), retrying transient 409 "host offline" across a ~16.5s budget (`_RETRY_DELAYS_S=(0.5,1.0,1.5,2.0,2.5,3.0,3.0,3.0)`, `:209`).

Also present: an orphan reaper (`_orphan_reaper_loop`, `connect.py:691`) sweeping every 2.0s (`_ORPHAN_REAP_INTERVAL_S`, `:144`) to reap grandchild tool processes (node/npm/chromium/tmux) left by a dead runner.

## 7. Host Identity & Daemon/Local Modes

`HostIdentity(host_id, name)` (`identity.py:39-51`), `host_id = "host_" + uuid4().hex`. Loaded/created from `~/.omnigent/config.yaml`'s `host:` section, or overridden entirely (no disk write) by `OMNIGENT_HOST_ID`+`OMNIGENT_HOST_NAME` env pair for managed sandboxes (`identity.py:28-29,77-85`). `OMNIGENT_HOST_TOKEN` carries tunnel auth, sent as `X-Omnigent-Host-Token`.

Daemon entry (`_daemon_entry.py:20-64`): parses `--server`/`--local` (mutually required), calls `run_host_process(server_url=...)` — same top-level function used by `omnigent host` directly. **Local mode** (`local_server.py`) does **not** implement a bespoke lightweight server — it spawns/manages the lifecycle of the *same* full `omnigent server` process bound to loopback (`_spawn_local_server`, `:584-666`, port 6767 default, `:694`), with a pidfile+config-signature check to reuse a healthy instance (`:442,90`). Local mode is architecturally identical to remote from the host daemon's perspective — it just also owns starting the server.

## 8. Git Worktrees (`host/git_worktree.py`)

All server-initiated via the frames in §2 (no autonomous scheduling), dispatched at `connect.py:1952-1957`. Naming: `<repo-parent>/<repo-name>-worktrees/<sanitized-branch>` (slashes→`-`), collision suffix `-2..-50` (`git_worktree.py:83-91,20-21`). Creation resolves the *main* worktree first via `git worktree list --porcelain` so new worktrees are always siblings not nested (`:373-379`), rejects existing branches (`:383-386`), runs `git worktree add -b <branch> <path> --end-of-options <base>` guarding against flag injection (`:392-399`). Removal resolves the main repo via `git rev-parse --git-common-dir`, runs `git worktree remove --force`, then optionally `git branch -D` in that order since git refuses branch deletion while checked out (`:429-462`). All git calls use argv lists (never shell), 120s timeout (`:16-18`).

## 9. Sandboxing

Wraps agent-spawned OS subprocesses/tool executions, not host/runner processes themselves. Shared interface `inner/sandbox.py`: `SandboxPolicy` (read/write roots, `allow_network`, dotfile-hiding, egress relay socket, `:53-120`), backend selected via agent YAML `os_env.sandbox.type` or platform default (`_default_sandbox_for_platform`, `:851`).
- **Linux bubblewrap** (`inner/bwrap_sandbox.py:183-243`, `type_name="linux_bwrap"`): requires `bwrap` on PATH or fails loud; cwd read-only unless `write_paths:["."]`; unshares network namespace unless `allow_network`.
- **macOS seatbelt** (`inner/seatbelt_sandbox.py:1-40,329-343`, `type_name="darwin_seatbelt"`): prepends `sandbox-exec -f <profile-file>` (SBPL), profile written to mode-0600 tempfile (not `-p` inline) to hide internals from `ps aux`; omits `mach-priv-host-port`/`iokit-open`; narrows `/dev` to read-only plus literal allows for `/dev/null,/dev/tty,/dev/dtracehelper` (`:18-28`).
Both registered at import time; callers include `inner/terminal.py:788,898`, `inner/acp_executor.py:235`, `tools/builtins/web_fetch.py:130-137`.

## 10. Key Class Relationships

- `HostProcess` (connect.py) owns: WS connection, `_unreported_exits`, spawned `Popen` handles, `_watch_runner`/orphan-reaper tasks. Creates: `HostLaunchRunnerResultFrame`, `HostRunnerExitedFrame` instances.
- `SessionResourceRegistry` (runner) owns `TerminalRegistry`; depends on external `HarnessProcessManager` for actual process spawning.
- `RunnerMcpManager`/`ProxyMcpManager` — interchangeable, created once per runner or per-session; consumed by `tool_dispatch.execute_tool`. `RunnerToolPolicyGate` constructed per `spec_hash`, cached on `RunnerMcpManager`.
- `pending_approvals` module-global dict — shared dependency of both local-policy ASK escalation and proxy-mode MRTR ASK.
- `RunnerRouter` (`routing.py`) — despite living in `runner/`, is server-side code (owns `TunnelRegistry`+`ConversationStore` refs), not instantiated in the runner process.
- `TunnelRegistry`/`WSTunnelTransport` — server-side, map `runner_id→WS`, frame server-originated httpx calls over it.

## Uncertainties (explicit)

- `designs/SESSION_GIT_WORKTREE.md` / `designs/SESSION_WORKSPACE_SELECTION.md` cited in code but absent from this checkout.
- Exact point where a tmux pane/OS process is forked for a harness lives in `omnigent.terminals.TerminalRegistry`/`HarnessProcessManager`, outside `runner/` — only the call boundary was confirmed, not internals.
- `RunnerRouter`'s presence in the `runner/` package despite being server-side code is unexplained — possibly a packaging artifact.
- Did not fully read `tool_dispatch.py`'s ~150 functions (260KB) — only `execute_tool`/`dispatch_tool_locally`/subagent paths verified; most `_execute_*_tool` REST/timer/browser handlers identified by signature only.
- `_ping_loop`/"3 missed pongs" logic inferred from a comment in `ws_tunnel/limits.py:9-10`; the actual loop body in `runner_tunnel.py`/`host_tunnel.py` wasn't read to confirm verbatim.
- Did not verify whether `HostRunnerExitedFrame` ever triggers automatic respawn elsewhere in server code outside `host_tunnel.py`.
- Full bwrap seccomp profile / SBPL rule set beyond cited excerpts unverified (large files, sampled not exhaustively read).
- TCP/UDS transports marked "Phase 2/3" in `transports/__init__.py:3-5`; unclear if any production path still selects them vs. `ws_tunnel` being the sole production transport.
