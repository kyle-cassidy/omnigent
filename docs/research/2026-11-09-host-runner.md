# Omnigent HOST + RUNNER — Research Report (2026-11-09 update)
> Produced 2026-11-09 by a read-only Claude research subagent (one of five parallel passes) updating the 2026-08-18 report. Method: read current code at commit b03ad1ac9, compared against the baseline report. Citations are file:line at that commit.

Baseline: `docs/research/2026-08-18-host-runner.md` (v0.4–0.6-era code). Current tree: v0.22.0.dev0, 6,206 commits on the branch. Change tags: **NEW / CHANGED / REMOVED / UNCHANGED**. Paths are relative to `omnigent/` unless they start with `server/` (also under `omnigent/`).

## 0. Headline deltas

- **Size.** `host/` went from ~4.9k to **17.8k lines** in 26 files. `runner/` went from ~31k to **73.6k lines** in about 60 files. The host grew from a launcher into a host *agent*: harness install and login, credentials, skills, plugins, MCP inventory, a file server for offline runners, and local-transcript import.
- **`runner/app.py` is no longer ~19k lines (CHANGED).** It is now 9,451 lines / 434KB. The per-harness `_auto_create_*_terminal` code moved to `runner/native/orchestration.py` (11,493 lines; its module docstring is a verbatim copy of app.py's). Routes moved to `mcp_routes.py`, `resource_routes.py` and `model_option_routes.py`. Policy proxying moved to `policy_proxy.py`. Interrupt handling moved to `native/interrupt.py`, and control events to `native_controls.py`. Net effect: the runner got larger, and app.py got smaller.
- **Host frame protocol: 22 kinds became 61 (CHANGED).** The baseline table listed 22 wire kinds (`HostFrameKind`, `host/frames.py:215-278`). The baseline text said "18-kind" but its own table listed more. The wire is still `frame_protocol_version=1` (`host/connect.py:5630`). New features are gated by **capability tokens** in the hello frame (`host/frames.py:73-108`), not by version bumps.
- **NEW: runner zygote forkserver.** One long-lived process imports the runner graph once and `os.fork()`s a runner per session (`runner/_zygote.py`, `host/runner_zygote.py`). It is on by default on POSIX. `OMNIGENT_RUNNER_ZYGOTE=0` turns it off, and the host falls back to `Popen` when the zygote fails.
- **CHANGED: runner tunnel.** It gained `event.ready/batch/ack` frames for bounded event ingestion (`ws_tunnel/frames.py:48-50`). The pong now carries `keep_warm` for the managed-sandbox keepalive (`frames.py:156-166`, `busy_check.py`). The runner hello advertises a loopback **direct-attach** port and token (`frames.py:88-89`, `direct_attach.py`).
- **CHANGED: host reconnect cap** cut from 10s to **3s** (`host/connect.py:531`). New: a recycle-streak limit, a loopback-refused fatal threshold, and separate open timeouts (`:540-560`).

## 1. Module inventory

**Host** (`omnigent/host/`, 17,809 lines):
| Path | Lines/KB | Tag | Responsibility |
|---|---|---|---|
| `connect.py` | 6393 / 292KB | CHANGED (×3.2) | `HostProcess` (`:1279`): the WS loop `run()` (`:4934`) → `_connect_and_serve()` (`:5284`) → `_handle_raw_message()` (`:5840`). It has about 40 `_handle_*` frame handlers (`:2021-4670`) and holds the zygote/`Popen` handles, the orphan reaper and the harness-login state |
| `frames.py` | 3755 / 142KB | CHANGED (×3) | 61-kind enum, ~55 dataclasses, capability tokens, encode (`:1645`), decode (`:2450`), chunked import assembler (`:2315-2450`) |
| `local_server.py` | 1299 / 55KB | CHANGED | Detached, *persistent* shared local `omnigent server`: pidfile, config signature, `/proxy/<port>` base path, default port 6767 (`:886`), `_spawn_local_server` (`:754`). The daemon is the only process that starts it (`:10-14`) |
| `git_worktree.py` | 795 / 34KB | CHANGED | Same naming scheme (`:419-450`). Adds create **time budgets** with rollback reserve (`:25-31,180-229`), `for_cleanup` listing and commit timestamps (`:308-331`) |
| `daemon_lifecycle.py` | 735 | NEW | `flock` on `<data-dir>/daemons/<hash>.json`. The record doubles as a liveness lock that is immune to PID reuse. The daemon kills itself if the record is deleted or claimed by another daemon (docstring `:1-17`; poll interval `connect.py:538`) |
| `crash_reporting.py` | 433 | NEW | Routes every host exit through a structured `host_exiting` event, and logs thread and asyncio-task crashes |
| `runner_zygote.py` | 418 | NEW | `ZygoteManager` / `ZygoteRunnerProc` (a drop-in for `Popen`; poll goes through the zygote because the zygote is the real parent) |
| `databricks_credential.py` | 378 | NEW | Managed sandbox: writes a host-only `~/.databrickscfg` profile with no token. Harnesses mint bearer tokens per use from the server's credential broker |
| `daemon_launch.py` | 359 | UNCHANGED (logic) | `launch_or_reuse_daemon_runner` (`:248`). Same `_RETRY_DELAYS_S` ≈16.5s budget (`:315`) |
| `harness_startup.py`, `identity.py` (351), `identity_env.py` | — | CHANGED/NEW | `identity_env` is a leaf module of env-var names (`HOST_TOKEN_ENV_VAR` `:20`), split out so the warm-pool probe stays cheap. `MANAGED_HOST_TOKEN_HEADER` is at `identity.py:39` |
| `mcp_inventory.py` (346), `mcp_tools.py` (315) | — | NEW | User-level MCP server inventory with secrets stripped (env, headers, args and URLs dropped). On-demand probe of one server's tools |
| `warm_bootstrap.py` | 344 | NEW | Binds an already-running warm sandbox Pod to one managed host through an activation file on a memory-backed volume |
| `service.py` (330), `service_entry.py` | — | NEW | Installs the host as a per-user launchd/systemd service |
| `maintenance.py` (279), `launch_audit.py`, `plugins.py`, `skills.py`, `skill_content.py`, `jcode_databricks.py` (231) | — | NEW | Machine-wide cleanup; launch-context audit join; Claude plugin, skill and skill-content inventory; session-private jcode gateway config |
| `_daemon_entry.py` | 264 | CHANGED | Daemon entry (it was 68 lines) |

**Runner** (`omnigent/runner/`, 73,650 lines; top items):
| Path | Lines | Tag | Responsibility |
|---|---|---|---|
| `native/orchestration.py` | 11493 | NEW (split from app.py) | 13 `_auto_create_*_terminal` functions: opencode `:1761`, pi `:2771`, cursor `:3258`, goose `:3562`, hermes `:3755`, kiro `:4026`, devin `:4167`, qwen `:4496`, kimi `:4781`, codex `:5201`, antigravity `:6904`, claude `:9089`, repl `:10111`. Also native launch-config snapshots and teardown |
| `tool_dispatch.py` | 9479 / 389KB | CHANGED (+48%) | `execute_tool` (`:6961`), `dispatch_tool_locally` (`:7368`) |
| `app.py` | 9451 / 434KB | CHANGED (−51%) | `create_runner_app` (`:1241`). Session core routes: `/health` `:2572`, `POST /v1/sessions` `:3496`, stream `:3541`, events `:7204`, shell `:8360`, reset-state `:8548`. Also `_build_spawn_env_from_spec` (`:9141`) |
| `native_controls.py` | 2669 | NEW | Native-harness `model_change`, `effort_change`, `compact` and cost-popup handling |
| `resource_registry.py` | 2621 | CHANGED (×1.9) | `SessionResourceRegistry` (`:551`): `launch_required_terminal` `:1425`, `launch_auxiliary_terminal` `:1458`, `_launch_terminal_with_lifecycle` `:1491` |
| `_entry.py` | 2428 | CHANGED (×2) | `create_app` (`:1540`), spec resolver (`:1486`), parent-PID watchdog (`:1151-1164`), `HarnessProcessManager` wiring (`:1566,1587`), `main` (`:2398`) |
| `github_resource.py` (2432), `pr_resource.py` (1271), `pr_observer.py`, `pr_reports.py`, `session_prs.py`, `github_review_threads.py`, `git_providers/` (GitHub, GitLab, Azure DevOps, local_git), `azure_devops_client.py`, `gitlab_client.py` | ~9k | NEW | Session PR panel and multi-provider PR tracking, mostly via the `gh` CLI and provider CLIs |
| `resource_routes.py` | 2117 | NEW (split) | Environments, terminals (+ WS attach `:962`), filesystem, search, changes, ~25 GitHub routes |
| `turn_routing.py` (1665), `subagent_routing.py` (1599) | — | NEW | Advisory model routing through loopback HTTP relays called by harness hooks (`UserPromptSubmit`, `PreToolUse`). Routing failure → the prompt or spawn runs unchanged |
| `subagent_work.py` (1347), `subagent_recovery.py`, `acp_subagent_sessions.py` | — | NEW | Async `sys_session_send` tracking, delivery to the parent inbox, recovery after restart, mirroring of ACP sub-agents into child sessions |
| `environment_filesystem.py` | 1634 | CHANGED (×2) | `CallerProcessFilesystem` over a sandboxed `OSEnvironment` |
| `native/interrupt.py` (907), `codex/goal.py` (379), `background_titles/` | — | NEW | `NativeInterruptRunner` (16 per-harness closures folded into one registry); Codex goal JSON-RPC; background session titling |
| `_zygote.py` | 882 | NEW | Forkserver child side |
| `mcp_manager.py` (894), `proxy_mcp_manager.py` (638) | — | CHANGED | Same split as before. `_POOL_SPEC_CAPACITY = 8` (`mcp_manager.py:84`). `ProxyMcpManager.call_tool` is at `proxy_mcp_manager.py:309` |
| `mcp_execution_registry.py` (185), `mcp_routes.py` (589) | — | NEW | MCP executions protected so a server-tunnel replacement can't cancel and replay a side-effecting call; reattach by op id and step |
| `launch_failure.py` (747) | — | NEW | One classifier for terminal exits (title/cause/remediation) across all ~20 harnesses |
| `policy.py` (507), `policy_proxy.py` (244), `pending_approvals.py` (372) | — | CHANGED | See §4 |
| `routing.py` (491) | — | UNCHANGED (role) | `RunnerRouter` is still **server-side** code inside `runner/` (imports `WSTunnelTransport`, `routing.py:20`) |
| `direct_attach.py` (296), `busy_check.py`, `sign_in_watch.py`, `slow_io.py`, `content_search.py`, `session_history.py`, `session_init_protocol.py`, `model_option_routes.py`, `native_file_observer.py` | — | NEW | See the sections below. `slow_io` provides dedicated thread pools so git/`gh` work cannot starve terminal delivery |

**Transports** (`runner/transports/`): `uds.py` (177) and `tcp.py` (172) are UNCHANGED and limited to tests. The docstring (`transports/__init__.py:1-20`) now says outright that local server-to-runner traffic uses the WS tunnel. `ws_tunnel/` is CHANGED: `serve.py` 2069 lines, `registry.py` 1144, `frames.py` 789, `transport.py` 215. **NEW** files: `event_delivery.py` (354) and `diagnostics.py` (369, timing data only; it does not drive liveness).

## 2. Host frame protocol (`host/frames.py`, enum `:215-278`)

All frames are JSON. Ping/Pong are reused from the runner tunnel. "Reply" means the server→host request frame has a matching `*_result` frame from host→server, correlated by `request_id`.
| Kind (`host.*`) | Dir | Purpose | Tag |
|---|---|---|---|
| `hello` (`:285`) | H→S | Registers the host. Adds `frame_protocol_version`, `harness_accounts`, `gateway_inference`, `interactive_shells`, `capabilities`, `connection_id` and `installation_id` (`:320-332`) | CHANGED |
| `connection_error` (`:336`) | S→H | Server channel failure (`stage`, `retryable`) sent before close | NEW |
| `harness_readiness` (`:350`) | H→S | Pushes changes to per-harness readiness/gateway status while the tunnel is open | NEW |
| `launch_runner` / `_result` (`:369/407`) | S→H / reply | Spawns a runner. Adds `inference_config`, `superseded_runner_ids` and `scheduled_run`. Error codes: `harness_not_configured`, `workspace_missing`, `launch_handler_failed` (`:56-67`) | CHANGED |
| `stop_runner` / `_result` (`:440/454`) | S→H / reply | Terminates a runner | UNCHANGED |
| `runner_exited` (`:470`) | H→S | Reports an unexpected death (exit code + redacted log tail after `RUNNER_LOG_TAIL_HEADING`) | UNCHANGED (+heading) |
| `runner_status` / `_result` (`:498/519`) | S→H / reply | Asks whether a runner process is alive, dead or unknown | NEW |
| `stat`, `list_dir`, `create_dir` (+results) (`:542-849`) | S→H / reply | Filesystem probes for the workspace picker | UNCHANGED |
| `create_worktree`, `remove_worktree`, `list_worktrees` (+results) (`:695-808`) | S→H / reply | Git worktrees. Create now takes `timeout_s` (cap `worktree_create_budget`) | CHANGED |
| `install_harness` / `_result` (`:874/895`) | S→H / reply | Installs a harness CLI on the host | NEW |
| `refresh_harnesses` / `_result` (`:935/942`) | S→H / reply | Re-probes harness readiness | NEW |
| `probe_harness_accounts` / `_result` (`:954/962`) | S→H / reply | Non-secret account metadata | NEW |
| `harness_login`, `_cancel` (+results), `harness_login_completed`, `harness_logout` / `_result` (`:972-1031`) | S→H / reply; `_completed` H→S | Codex device-code login flow and logout | NEW |
| `store_secret` / `_result` (`:1042/1087`) | S→H / reply | Writes a provider credential on the host | NEW |
| `detect_credentials` / `_result` (`:1118/1133`) | S→H / reply | Lists credentials already on the host that can be adopted | NEW |
| `fs_request`, `fs_write_request` → `fs_result` (`:1150/1180/1219`) | S→H / reply | Serves the web file panel (list, read, diff, search, GitHub PR info) and workspace edits from the host **when the runner is offline** | NEW |
| `model_options` / `_result` (`:1244/1252`) | S→H / reply | Model choices before launch | NEW |
| `skills`, `skill_content`, `plugins` (+results) (`:1271-1360`) | S→H / reply | Skill, skill-content and Claude plugin inventory | NEW |
| `harness_startup` / `_result` (`:1300/1308`) | S→H / reply | Host defaults for a native launch | NEW |
| `mcp_servers`, `mcp_tools` (+results) (`:1316-1381`) | S→H / reply | User-level MCP inventory; on-demand tools probe | NEW |
| `import_local`, `import_local_by_id` (`:1417/1439`) | S→H | Reads recent local harness transcripts | NEW |
| `import_local_session`, `_session_chunk`, `import_local_done` (`:1456/1474/1498`) | H→S (stream) | Streamed, chunked transcript import (`ImportLocalSessionChunkAssembler` `:2355`) | NEW |

Capability negotiation is NEW. The host advertises `HOST_CAPABILITIES` (`:93-108`), and the server advertises its own in the `X-Omnigent-Server-Capabilities` upgrade header (`:112`). For example, `tmux_readiness` lets the host send `"tmux-unavailable"` readiness instead of mapping it down to `True`.

## 3. Runner lifecycle

```mermaid
sequenceDiagram
  participant S as Server
  participant H as Host daemon (connect.py)
  participant Z as Zygote (_zygote.py)
  participant R as Runner (_entry/app.py)
  participant T as Harness (tmux/TUI)
  H->>S: WS /v1/hosts/{id}/tunnel · host.hello(capabilities, runners[])
  S->>H: host.launch_runner(binding_token, workspace, harness)
  H->>Z: {"cmd":"fork", env, log_path} (fallback: Popen -P -m omnigent.runner._entry)
  Z-->>R: os.fork() → _entry.main()
  R->>S: WS /runners/{runner_id}/tunnel · hello(direct_attach_port/token)
  H-->>S: host.launch_runner_result
  S->>R: request frames (POST /v1/sessions, /events …) over runner tunnel
  R->>T: _auto_create_<harness>_terminal (native/orchestration.py)
  R->>S: event.batch → event.ack ; pong(keep_warm)
  Note over H,R: crash → _watch_runner → host.runner_exited (parked in _unreported_exits if tunnel down)
```

- **Spawn (CHANGED).** `_handle_launch` (`connect.py:2021`) → `_handle_launch_impl` (`:2041`). It tries `_ensure_zygote_started` (`:2334`) first. When that fails it falls back to `subprocess.Popen([sys.executable, "-P", "-m", "omnigent.runner._entry"], cwd=workspace, stdin=/dev/null)` (`:2439-2450`). `-P` was added so a workspace that is itself an omnigent checkout cannot shadow the installed package. The env comes from the allowlist `_build_runner_env` (`:953`). The zygote prestarts in the background at `run()` (`:4986-4989`).
- **Zygote protocol (NEW)** (`_zygote.py:20-44`): newline-delimited JSON over an AF_UNIX socketpair with `ping`, `fork` and `poll`. Parent-PID contract: `RUNNER_PARENT_PID` is set to the zygote's pid. When the daemon dies, the socket hits EOF, the zygote exits, and the runners are reparented; their `getppid()` watchdog (`_entry.py:1151-1164`) then tears them down.
- **Watch, stop and status.** `_watch_runner` (`:2710`) still polls every 0.5s (`:379`), and a new `_watch_runner_connect` (`:2803`) also watches the connect step. `_handle_stop` is at `:2502`, and the new `_handle_runner_status` at `:2662`. Reporting an exit is UNCHANGED: when the tunnel is down, the report is parked in `_unreported_exits` (`:1405,2874`) and flushed after reconnect (`:5657-5658`). Server ingest: `server/routes/host_tunnel.py:511` (`runner_exit_reports`).
- **Orphan reaping (CHANGED).** The reaper now starts only when `_install_child_subreaper()` succeeds (`:396-415,4954-4957`). That needs Linux `PR_SET_CHILD_SUBREAPER`, so macOS hosts do not run the grandchild reaper.
- **Daemon self-retirement (NEW).** `daemon_lifecycle` flock plus a monitor poll every 60s (`connect.py:538`).
- **Spec resolution (CHANGED).** `_resolve_agent_spec_from_server` (`_entry.py:1486`). `X-Agent-Session-Scoped` now **defaults to `"true"`** (tenant-scoped, so no `${VAR}` expansion) and only an explicit `"false"` expands (`_entry.py:1451-1452`). The fail-safe direction is the same, but the implementation changed. The bundle cache key now includes a SHA-256 of the content (`:1454-1456`).

## 4. Tool dispatch, MCP, policy and approvals

- Dispatch entry points moved but keep the same names: `execute_tool` (`tool_dispatch.py:6961`) and `dispatch_tool_locally` (`:7368`). The MCP execute route is now `mcp_routes.py:111` (it was `app.py:18151`).
- Policy over the server round-trip **moved** to `policy_proxy.py:52` (`_evaluate_policy_via_omnigent`, previously `app.py:6430`). Semantics are UNCHANGED: `FAIL_CLOSED_PHASES` → DENY, otherwise ALLOW (`:104-109`). NEW: comments at `:23-31` document that an ASK gate is read without a short timeout, after a 30s cut-off once fail-closed parked approvals to DENY (guarded by `tests/test_ask_timeout.py`).
- Local fast path: `RunnerToolPolicyGate` (`policy.py:169`), `_evaluate_policies` (`:397`). NEW: `AgentStartPolicyError` (`:124`) and an agent-start gate (`app.py:9387`).
- `pending_approvals.wait_for_user_approval` is now at `:332`, with `resolve` at `:216`. The module-global Future registry is UNCHANGED in role.
- `mcp_execution_registry` (NEW) lets a reconnected server *reattach* to an in-flight MCP call instead of replaying it.

## 5. Transports

- **UNCHANGED topology.** Each runner dials its own WS straight to the server (`serve.py:1177`; server route `server/routes/runner_tunnel.py:430`), and the host has a separate WS (`host_tunnel.py:225`, URL built at `connect.py:1656`). The host never carries runner traffic. The binding token, `token_bound_runner_id` (`runner/identity.py:32,200`) and stripping from child env (`:94-97,174`) are all UNCHANGED.
- **Runner `FrameKind`** (`ws_tunnel/frames.py:30-50`): `hello`, `request`, `response.head/body/end`, `request.cancel`, `ping`, `pong`, `ws.open/frame/close`, plus **NEW** `event.ready`, `event.batch` and `event.ack`. Native-forwarder events now ride the existing tunnel in batches of at most 32 events / 256KB, with at most 34 pending batches (`event_delivery.py`, constants near the top). The server ingests them per PR #8244.
- **Keepalive** is UNCHANGED: 30s ping, 90s timeout, now in `util/tunnel_limits.py:25-26`. The max message size is 100MB (`:14`). Pong carries `keep_warm` (NEW).
- **Runner reconnect** (`serve.py:103-138`): base 0.5s, cap 10s, ±50% jitter, and a recycle window of 0.5–3s for planned restarts.
- **Host reconnect** (CHANGED, `connect.py:530-560`): cap **3.0s** (was 10s). Open timeout is 10s on the first connect and 3s on reconnects. `_RECYCLE_PROMPT_MAX_STREAK=10`. `_MAX_CONSECUTIVE_AUTH_ERRORS=3` applies to fresh hosts only. `_LOOPBACK_REFUSED_FATAL_ATTEMPTS=100` (~5 min) before exiting when the local server is gone. `_LOGIN_REDIRECT_FATAL_ATTEMPTS=3` is still there (`:926`). Explicit recycle close codes 1001/1012 are at `:5086-5130`.
- **Direct attach (NEW)** (`direct_attach.py`): the runner serves the terminal-attach handler on `127.0.0.1` with a per-process bearer token and an Origin check. The server passes the URL only to `LEVEL_OWNER` callers. Same-machine browsers skip both WAN legs, and the web client falls back to the relay when the listener can't be reached.

## 6. Persistence

- The host persists only small files: the `~/.omnigent/config.yaml` host identity (or managed-sandbox env), the daemon registry record plus flock (`daemon_lifecycle`), the local-server pidfile and config-signature sidecar (`local_server.py:109-123,362,418`), per-runner log files, the per-session inference config (`connect.py:1069`), and the Databricks profile/sidecar (managed sandboxes only).
- The runner holds **no database**: no `sqlite` imports anywhere in `host/` or `runner/`. Its state lives in memory. Restart resilience comes from the server: undrained sub-agent results (`subagent_recovery.py`), history replay into the harness (`session_history.py`), turn-replay recovery (`orchestration.py:856`), and the on-disk agent-bundle cache keyed by version+digest (`_entry.py:1454`).

## 7. Sandboxing (`inner/`, outside these packages; for context)

`inner/sandbox.py` is 1516 lines; default backend `_default_sandbox_for_platform` is at `:1440`. Backends: `linux_bwrap` (`bwrap_sandbox.py:266`), `darwin_seatbelt` (`seatbelt_sandbox.py:343`, now 2224 lines), and **NEW `windows_jobobject`** (`windows_jobobject_sandbox.py:177`). The Windows backend gives process-tree lifetime and resource limits only, with no filesystem or network isolation (`:1-12`). Also **NEW**: `sandbox/copy_on_write.py`, Linux overlay mounts owned by the environment. Managed cloud sandboxes do **not** live in host/runner. They are in `onboarding/sandboxes/` (agent_sandbox + warm pool, blaxel, boxlite, cwsandbox, databricks, daytona, e2b, gensee, islo, kubernetes, lambda_microvm, microsandbox, modal, openshell) and `server/managed_sandbox_reaper.py`. The host-side hooks are `warm_bootstrap.py`, `identity_env.py` and `databricks_credential.py`. No "Arclet" or "Arca" modules were found.

## 8. Dependencies

- Host → `omnigent.harness_accounts`, `harness_availability`, `onboarding.harness_readiness`, `workspace_fs.WorkspaceReader` (for `fs_request`), `inner.native_attachments`, `runner._zygote` (env-var constant only), `websockets`, `pydantic`.
- Runner → `runtime.harnesses.process_manager.HarnessProcessManager` (`_entry.py:1566`); the tmux/PTY fork is still outside `runner/`. Also `harnesses.*_native` bridges (imported lazily), `spec.load`, `inner.*` sandbox/OS env, `gh` / `glab` / `az` CLIs, FastAPI, httpx, websockets.
- Server → `runner.routing.RunnerRouter` and `ws_tunnel.registry/transport`, which are imported **by the server** even though they live in `runner/`.

## 9. Baseline claim check

| Baseline claim | Status |
|---|---|
| app.py 883KB/19,178L, `create_runner_app` at :7880 | **Stale** — 434KB/9,451L, `:1241` |
| tool_dispatch 6,403L, `execute_tool` :4389 | Stale — 9,479L, `:6961` |
| connect.py 2,014L; reconnect cap 10s | Stale — 6,393L; cap 3.0s |
| frames.py "18-kind" enum | Stale — 61 kinds (the baseline table actually listed 22) |
| Runner owns its own WS; host doesn't proxy | **Holds** |
| Binding token → SHA-256 runner_id, stripped from children | Holds (line numbers moved) |
| `_RETRY_DELAYS_S` ≈16.5s | Holds (`daemon_launch.py:315`) |
| Worktree naming `<repo>-worktrees/<branch>` | Holds; budgets added |
| `HostRunnerExitedFrame` purely diagnostic | Holds in host_tunnel; server-wide not reverified |
| Orphan reaper every 2s, unconditional | **Changed** — gated on Linux subreaper (`:4954`) |
| MCP LRU capacity 8 | Holds (`mcp_manager.py:84`) |
| Runner launch via `Popen -m runner._entry` | Changed — zygote fork first, `Popen -P` as fallback |
| TCP/UDS "Phase 2/3" ambiguous | Resolved — test-only per docstring |

## Uncertainties

- Did not read `native/orchestration.py` (11.5k lines) or `tool_dispatch.py` (9.5k) beyond signatures, so the per-harness launch internals are unverified.
- Frame directions for result frames are inferred from naming and docstrings. Not every decoder pair in `_decode_known_host_frame` (`frames.py:2495`) was traced to a sender.
- The baseline's own frame count was internally inconsistent ("18-kind" vs a 22-kind table), so the "22→61" delta assumes the table.
- Whether the server now auto-respawns runners on `runner_exited` or via `runner_status` (NEW) was not checked in server code.
- `RUNNER_PARENT_PID` semantics are from the `_zygote.py` docstring. The non-zygote `Popen` path presumably sets it to the daemon pid, but this was not traced.
- Line counts for the "~4.9k / ~31k" baseline totals are reconstructed from the baseline tables, not from a checkout of the old commit.
- Arclet/Arca: no matches in the tree. They may be server-side names under a different spelling, or they may not exist.
- The sandbox section is context only. bwrap/seatbelt profile internals and `copy_on_write.py` were not reread.
