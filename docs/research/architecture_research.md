What it is. Omnigent calls itself a "meta-harness": one orchestration layer that sits above coding agents — Claude Code, Codex, Cursor, OpenCode, Hermes, Pi, plus custom YAML-defined agents — so you can start a session from any device (terminal, browser, phone, desktop app), mix agents in one session, and put policies in front of them (spend caps, approval gates, tool restrictions).

The three roles. It splits into server, host, and runner:

- Server (omni.delta-dms.com for you) is the coordination brain: it holds sessions, auth, and the web/phone UIs. It never touches your files directly.
- Host is what your new LaunchAgent keeps alive. Reading host/connect.py: it opens an outbound WebSocket to the server, registers itself (studio.local), then just listens for frames — host.launch_runner, host.stop_runner, plus a small filesystem vocabulary (list a directory, stat a file, create/list/remove git worktrees). It's deliberately dumb: a doorman that spawns things on demand and reports back.
- Runner is what the host spawns per session — a little FastAPI app that "owns harness subprocesses." It resolves which agent you asked for, launches the actual claude/codex/etc. process, and mediates everything: tool dispatch, MCP servers, pending approvals, policy enforcement. Those hash-named directories you'd see in ~/.omnigent/codex-native/ are per-session state for these, with a process-registry.json tracking what's alive.

▎ In plain terms: the server is the switchboard, your Mac Studio is a phone that's always off the hook waiting for calls, and each call spins up a fresh operator (runner) who actually does the work. When you tap "new session" on your phone, the server sends one frame down that WebSocket, your host forks a runner, the runner boots Claude Code in a git worktree, and the transcript streams back up to whatever screen you're on.

Why the dependencies I mentioned:
- node/npm — the harnesses themselves (Claude Code, Codex, OpenCode, Pi) are npm-installed CLIs; omnigent installs and launches them.
- git — each session can get its lel agent sessions on the same repo don't stomp on each other. The host has dedicated frames just for worktree lifecycle.
- tmux — the "native" wrappers (om treat each agent as aterminal-first program: they run the real TUI inside tmux and attach your TTY to it over the
terminal WebSocket protocol. That'ession "follow you" — the TUI keeps living in tmux on your Mac while you detach and reattach from another device. On Linux each
of those terminals also gets a bubcOS it uses the built-in seatbeltsandbox.

The design choice worth admiring is the outbound-only tunnel. Your Mac Studio never opens a
port; it dials out to the server a which is why it works behind yourhome NAT with zero firewall config — the same pattern as your Cloudflare tunnel for delta-dms.
It's also why the LaunchAgent mattWebSocket is your machine'spresence in the system. No connection, no host, no runners — which is exactly what the
host=offline status was telling us

If you want to go deeper, the repoES.md for the governance layer anddesigns/RUNNER.md for the runner contract are probably the two best reads.

✻ Cooked for 1m 21s · 1 monitor still running

❯ Wonderful—let’s go deep. Please create a polished, visually rich HTML architecture
  walkthrough that I can explore and
  I’d like it to include:
  An overall system architecture diades, services, data stores,agents, processes, and external interfaces.
  A clear explanation of whether thi“one authoritative node, manyprojections” model. Identify the system of record, each projection or derived view, and the
  direction and mechanism of synchro
  Component-level diagrams for each major subsystem, showing its responsibilities, inputs,
  outputs, persistence, dependencies
  A dedicated explanation of tmux’s role: what work it coordinates, what processes or agents
  run in its panes/sessions, how cron works, and whether tmux is onlya terminal multiplexer or an essential coordination layer.
  A UML-style class diagram or codebring the main classes, interfaces, modules, and services. Show:
  inheritance and interface implemen
  composition and ownership;
  direct dependencies;
  important data/control-flow relationships;
  key public responsibilities for ea
  A few sequence diagrams for the most important workflows—especially initialization, synchronization, updates to the aution to projections, failure/retry behavior, and conflict handling.                                                             A concise narrative that distingui
  the source of truth;                                                                         durable state versus derived state
  synchronous versus asynchronous communication;                                               orchestration versus business logi
  runtime topology versus code-level structure.                                                Please make the page visually strirogressive disclosure, clearlegends, clickable or expandable details where helpful, and diagrams that remain readable at both overview and implementation dtions or uncertainties rather than inventing details. feel free to use subagents, opus/sonnet for decomposition and a faster    turnaround
                                                                                               Excellent — this deserves the fullresearch agents over the omnigentcodebase in parallel (one per subsystem area), then synthesize their findings into a polishedinteractive HTML walkthrough. Launw.
                                                                                               Explore(Map omnigent server subsys
                                                                                               Explore(Map omnigent host+runner s
                                                                                               Explore(Map native wrappers and tm
                                                                                               Explore(Map classes and key workfl
                                                                                               Four researchers are running. While design guidance for the artifact.
                                                                                               Skill(artifact-design)
Successfully loaded skill                                                                 
⏺ Skill(artifact-diagramming)                                                                  Successfully loaded skill
                                                                                               Design guidance loaded. The four rrunner, tmux/native wrappers, class map + workflows) are still digging through the codebase — I'll synthesize their reports into the walkthrough page as soon as th

※ recap: I'm building an interactiverough of omnigent for your review.Four research agents are mapping the codebase (server, host/runner, tmux wrappers, class map).
  Next: synthesize their reports intsable recaps in /config)


@ class-workflow-mapper❯ Omnigent class map + workflow traces, cross-cutting areas
  Research done. I focused on cross- spec, policies, entities/sessions, MCP tooling, harness abstraction) rather than duplicating what
  host-runner-mapper/server-mapper lhost/runner/server internals — letme know if you need me to go deeper there too.

  CLASS MAP

  Spec / Agent definition (omnigent/spec/)

  - AgentSpec — omnigent/spec/types.py:1382-1546. The fully parsed agent image. Fields: llm:
  LLMConfig, tools: ToolsConfig, exedrails: GuardrailsSpec|None,mcp_servers: list[MCPServerConfig], sub_agents: list[AgentSpec] (recursive), os_env,
  terminals, skills: list[SkillSpec]scriminator ("omnigent" default,"claude_sdk", "agents_sdk"); for "omnigent" the concrete harness
  (claude-sdk/codex/codex-native/etcig["harness"], exposed viaExecutorSpec.harness_kind property (types.py:599-616).
  - ExecutorSpec — types.py:487-616.ApiKeyAuth|DatabricksAuth|ProviderAuth, timeout, max_iterations.
  - LLMConfig — types.py:644-708. moetryPolicy, fallback_models.
  - RetryPolicy — types.py:42-254. Layered retry (L0 SDK-internal via per-provider adapter
  properties .openai/.anthropic/.cla2 workflow-level viacompute_backoff_delay).
  - GuardrailsSpec — types.py:1358-1elDef], policies: list[PolicySpec], ask_timeout (default DEFAULT_ASK_TIMEOUT=86400, types.py:39).
  - PolicySpec (base) / FunctionPoli.py:1291-1354. on:list[PhaseSelector], condition (label-gate), function: FunctionRef.
  - MCPServerConfig — types.py:861-9io", redacting __repr__.
  - Parser/validator: omnigent/spec/parser.py, omnigent/spec/validator.py (not read in depth —
  grounded from types.py docstrings

  Policies (omnigent/policies/, runt not read:omnigent.runtime.policies.engine.PolicyEngine)

  - Policy (ABC) — omnigent/policies/base.py:32-96. spec: PolicySpec; abstract async
  evaluate(ctx, context) -> PolicyRet_turn().
  - FunctionPolicy — omnigent/policies/function.py (implements Policy, resolved via
  resolve_function_policy).
  - PromptPolicy referenced in base.py docstring as sibling of FunctionPolicy but I did not open
  prompt.py.
  - EvaluationContext (frozen dataclass) — omnigent/policies/types.py:68-203. phase, content,
  tool_name, trajectory, actor, sessharness, labels, llm_client.
  - PolicyResult (frozen dataclass) — types.py:207-276. action (ALLOW/ASK/DENY), reason,
  set_labels, deciding_policies, datate_updates.
  - ElicitationRequest — types.py:280-331. MCP-elicitation-shaped ASK payload (message,
  requestedSchema, policy_names).
  - PolicyLLMClient — types.py:335-475. Pre-bound LLM client for policy callables, with serial
  fallback-model retry on .create().
  - Runner-side mirror: RunnerToolPolicyGate — omnigent/runner/policy.py:109-306. Re-implements
  TOOL_CALL/TOOL_RESULT-only enforcer (constructed per spec_hash via.from_spec()) so MCP tool dispatch doesn't round-trip to the server for the fast ALLOW/DENY
  path; ASK escalates back to the ses. PolicyVerdict (frozen dataclass, policy.py:65-101) is its return type: action: "allow"|"deny"|"ask".

  Executor / Harness abstraction (omnigent/inner/executor.py)

  - Executor (base class, NOT ABC — default no-op methods) — executor.py:518-596. Key method:   async run_turn(messages, tools, sysyncIterator[ExecutorEvent] (raises NotImplementedError in base). Capability flags: supports_streaming(),                         handles_tools_internally(), supporsupports_tool_boundary_interrupt(), supports_stepwise_internal_turns(). Lifecycle: close_session(), interrupt_session(),          enqueue_session_message(), close()
  - Subclasses live in omnigent/inner/*_executor.py — one per harness family                    (claude_sdk_executor, claude_nativr, codex_native_executor,cursor_executor/cursor_native_executor, goose_executor/_native, hermes_executor/_native,      kimi_executor/_native, kiro_nativee_executor, pi_executor/_native,qwen_executor/_native, acp_executor, antigravity_executor/_native, databricks_executor,       openai_agents_sdk_executor). Each  (e.g. claude_sdk_harness.py) — Idid not open these; naming strongly implies Harness = process/session lifecycle wrapper,      Executor = the run_turn adapter Se I did not confirm there is asingle Harness base class/protocol — designs/harness-plugin-interface.md and                  designs/harness-capabilities-bench define this contract; I did notread them. Recommend a follow-up pass on those two files plus one *_harness.py file before    finalizing this part of the diagra
  - ExecutorEvent family (all dataclasses, executor.py:96-260): TextChunk, ReasoningChunk,      ToolCallRequest, TurnComplete, TooComplete, TurnCancelled,ExecutorError. ToolCallStatus enum (SUCCESS/ERROR/BLOCKED/CANCELLED).                         - MockExecutor — executor.py:604-6ing Executor.
                                                                                                Entities (omnigent/entities/)
                                                                                                - Conversation (dataclass) — entit7. THE session/conversation record: id, agent_id, runner_id, host_id, labels, session_state, session_usage, model_override,       harness_override, sub_agent_name, chived. runner_id = "hard affinity" per (missing) designs/RUNNER.md §5 — once set, every dispatch routes to that runner while     online.
  - ConversationItem / NewConversationItem (pydantic BaseModel) — conversation.py:661-765. Typedtranscript items; ItemData discrim, FunctionCallData,FunctionCallOutputData, ErrorData, ReasoningData, CompactionData, NativeToolData,             ResourceEventData, RoutingDecisionerminalCommandData. .to_api_dict()flattens for the wire API.                                                                    - Agent (dataclass) — entities/agegent record: id, bundle_location(content-addressed), version.                                                                 - LoadedAgent — agent.py:41-52. spPath, returned by AgentCache.load() (in omnigent/runtime/agent_cache.py, not opened).                                             
  MCP tooling (omnigent/tools/mcp.py, omnigent/runner/mcp_manager.py)                           
  - McpServerConnection (dataclass) — tools/mcp.py:392-1085. Owns one MCP server's lifecycle on a dedicated asyncio task (_run_lif for anyio cancel-scope-identitycorrectness. connect()→discovers tools (cached via module-level _discovery_cache TTLCache,     5min TTL); call_tool()→_call_tool_ential backoff; _CircuitBreaker(tools/mcp.py:211-323, trips after 5 consecutive failures, 30s cooldown, half-open probe)      wraps every call. Supports MRTR elRequired,call_tool_with_elicitation). Transports: stdio (unsandboxed subprocess) or HTTP                (Streamable-HTTP-first, SSE fallba
  - RunnerMcpManager — omnigent/runner/mcp_manager.py:203-787. Server-side (runner-side) pool    that shares McpServerConnections apec_hash (compute_spec_hash,mcp_manager.py:87), with _SharedServerEntry/_SpecEntry ref-counting and eviction.              prewarm(spec), schemas_for(spec),
                                                                                                 CLI / Host
                                                                                                 - CLI entry: omnigent/cli.py, Clic=_OmnigentCLI) at cli.py:1285).omni host group at cli.py:6995-7102 — the host() command IS the daemon (foreground); no server URL ⇒ spawns/reuses a local servert_server(), else does Databrickssign-in preflight (_ensure_databricks_server_auth) then calls                                  run_host_process(server_url=servert.py).
  - HostIdentity (dataclass) — omnigent/host/identity.py:40-51. host_id (host_{uuid4hex}), name. load_or_create_host_identity() (ids ~/.omnigent/config.yaml, with an env-var override path for server-managed sandbox hosts (OMNIGENT_HOST_TOKEN/_ID/_NAME).        - HostProcess — omnigent/host/conn37). Owns the WS tunnel clientloop, self._runners: dict[str, _RunnerHandle], reconnect state.                                - Host↔server frames — omnigent/hond enum (frames.py:37-57: HELLO,LAUNCH_RUNNER(+RESULT), STOP_RUNNER(+RESULT), RUNNER_EXITED,                                   STAT/LIST_DIR/CREATE_WORKTREE/REMOS/CREATE_DIR (+RESULT)).Dataclasses: HostHelloFrame (64-89), HostLaunchRunnerFrame (93-119),                           HostLaunchRunnerResultFrame (123-1e (180-200, one-way, no resultframe).                                                                                        - HostRegistry (server-side) — omny.py:223. Thread-safe(threading.RLock) in-memory map of live host WS connections per replica;                       register()/deregister()/get()/send asyncio.Queue, NOT thread-safeacross event loops — must be called on the host WS's owning loop).                             - RunnerToolDispatch etc. in omnigent/runner/tool_dispatch.py — notopened; app.py (6900+ lines) is dominated by per-harness _auto_create_<harness>_terminal       functions, strongly suggesting the FastAPI app module rather than aclean class hierarchy (UNCERTAINTY: no obvious Runner class — appears to be a module-level     FastAPI app + free functions; confer's findings).
  - RunnerToolPolicyGate — see Policies above.                                                   - pending_approvals module (not a pending_approvals.py.Process-global dict[str, asyncio.Future[bool]] registry                                        (register/resolve/cleanup/wait_for_approvals.py:133-198, default86400s wait).                                                                                 
  RELATIONSHIPS                                                                                 
  - AgentSpec --owns--> ExecutorSpec, LLMConfig, ToolsConfig, GuardrailsSpec,                    list[MCPServerConfig], list[SkillSb_agents, recursive).
  - GuardrailsSpec --owns--> list[PolicySpec] (base) / FunctionPolicySpec (subclass) --owns-->   FunctionRef.
  - FunctionPolicy --implements--> Policy (ABC).                                                 - RunnerToolPolicyGate --creates/o --wraps--> FunctionPolicy (viaresolve_function_policy); built .from_spec(AgentSpec).                                         - RunnerToolPolicyGate.evaluate_to>Policy.evaluate(EvaluationContext, ...) --returns--> PolicyResult --composed into-->           PolicyVerdict.
  - Executor (base) --implemented by--> *Executor subclasses in omnigent/inner/ (one per         harness). Executor.run_turn --yielasses.
  - LoadedAgent --owns--> AgentSpec + workdir: Path; produced by AgentCache.load() (not read).   - Conversation --references--> age (foreign-key-style string ids, not object refs) — DB-flavored entity, not an OOP composition.                                     - ConversationItem --owns--> ItemDof 11 pydantic models) via datafield; NewConversationItem/ConversationItem both validate type matches data's class            (_validate_type_matches_data).
  - McpServerConnection --owned/shared by--> RunnerMcpManager (ref-counted via                   _SharedServerEntry); RunnerMcpManaMcpServerConnection.call_tool--calls--> _call_tool_with_reconnect --uses--> _CircuitBreaker.                                - HostProcess --creates/owns--> suopen) per launched runner, keyed by runner_id (derived from token_bound_runner_id(binding_token)).                                 - HostRegistry (server) --tracks--ected HostProcess (client); server--sends--> HostLaunchRunnerFrame over the WS the HostRegistry tracks --host replies-->         HostLaunchRunnerResultFrame.
  - PolicyLLMClient --wraps--> omnigent.llms.client.Client (not opened) with server-level        model/connection pre-bound; used ilm_client for PromptPolicy-stylecallables.                                                                                    
  WORKFLOW TRACES                                                                               
  (a) Host initialization / auth / WS connect / HostHello / registered                          
  1. omnigent host [--server URL] invoked → omnigent/cli.py:7009 host(). If no --server,         resolves from config; if still emp server viaensure_local_omnigent_server() (cli.py:7065).                                                  2. If remote + Databricks-fronted,re_databricks_server_auth(server,non_interactive=...) (cli.py:7086) — runs the same browser login flow as omnigent login unless --non-interactive.
  3. run_host_process(server_url=server) (cli.py:7087) → omnigent/host/connect.py constructs a   HostProcess and calls .run() (conn
  4. HostProcess.run() (connect.py:1612-1708): starts an orphan-reaper background task, then     loops await self._connect_and_servhat classifies disconnects andbacks off exponentially (backoff = min(backoff*2*(1+jitter), _RECONNECT_CAP_S),                connect.py:1694) — except explicit12/1001) or ingress bounces(502/no-close-frame on non-loopback) which get a fast fixed-_RECONNECT_BASE_S retry            (connect.py:1674-1681). HostConnecuth) is NOT retried, propagatesout.                                                                                           5. _connect_and_serve() (connect.ppgrade headers via_build_connect_headers() (connect.py:1774-1825) — either the managed-sandbox                   X-Omnigent-Host-Token header, or as bearer token via_make_auth_token_factory (from omnigent/runner/_entry.py). Opens                               websockets.asyncio.client.connect(eaders,ping_interval=TUNNEL_KEEPALIVE_PING_INTERVAL_S, ...). Fatal upgrade errors                     (InvalidURI/InvalidStatus) are clae_error — permanent failures raiseHostConnectError and skip retry.                                                               6. _serve_frames(ws) (connect.py:1oFrame (version,frame_protocol_version=1, name, runners=self._alive_runner_ids(),                              configured_harnesses=configured_haelemetry_opt_out) —connect.py:1847-1859. Prints the "✓ Connected" banner (connect.py:1871). Then loops ws.recv()  with a 60s timeout, dispatching ea
  7. Server side: receives host.hello over the host tunnel router (create_host_tunnel_router,    server/app.py ~2358) → registers vhost_id, ws, hello, owner)(host_registry.py:238-274) — replaces+poisons any stale prior connection for the same host_id. 8. Login-flow branch: if the servee to a login page(unauthenticated), _login_redirect_streak increments; run() classifies this as a permanent     HostConnectError after enough consces "run omnigent login" ratherthan retrying forever (connect.py referenced around 1638-1651, exact streak-count threshold    not read in full).
                                                                                                 (d) Policy/approval flow — harnessding approval → user approves →tool proceeds                                                                                 
  1. Harness (via its Executor) emits ToolCallRequest (executor.py:134-146).                     2. Runner-local fast path: RunnerTol_call(tool_name, arguments)(runner/policy.py:159-182) builds an EvaluationContext(phase=Phase.TOOL_CALL, ...) and walks   _GatedPolicy entries via _evaluate306): eachFunctionPolicy.evaluate() is awaited; DENY short-circuits immediately; ASK is recorded but     evaluation continues (a later poliDENY); ALLOW/no-match continues. Araised exception in a policy callable is treated as fail-closed DENY (policy.py:263-279).      3. On PolicyVerdict(action="ask", s tool_dispatch, not directlyopened) escalates by POSTing evaluate_policy=True to the Omnigent server — per policy.py:20-24 docstring, the runner's local gateon channel, only the server does(it needs ConversationStore + can render the approval UI).                                     4. Server independently re-evaluat inomnigent.runtime.policies.engine, not opened) and parks an ElicitationRequest                  (policies/types.py:280-331) — thisding approval" badge reflects.
  5. Runner awaits the verdict: pending_approvals.wait_for_user_approval(elicitation_id,         conversation_id, publish_event, tipprovals.py:133-198) — registers an asyncio.Future[bool] (register(), pending_approvals.py:79-94), marks the session "awaiting     approval" via _session_pending (so_send injection from a parent agent doesn't perturb the parked turn — pending_approvals.py:51-60), and asyncio.wait_for(fut,       timeout=effective_timeout) (defaulan ASK is meant to survive the user stepping away).                                                                                6. User approves from any connecteo /v1/sessions/{id}/events(referenced in pending_approvals.py:5) → runner's session-event handler calls                  pending_approvals.resolve(elicitatending_approvals.py:110-130), which sets the Future's result.                                                                      7. wait_for_user_approval returns ry exit path — verdict, decline,timeout, or cancellation) it publishes response.elicitation_resolved so the server's           pending-elicitations index (and thstep(pending_approvals.py:186-197).                                                                8. Tool proceeds — the caller usesither dispatch the tool call forreal or synthesize a denial via format_deny_text(policy_name, reason) (policy.py:314-336),     which the LLM sees as the tool's o: <name>] {...}").
  9. Fail-closed phases: FAIL_CLOSED_PHASES = ("PHASE_TOOL_CALL", "PHASE_REQUEST")               (policies/types.py:64) — an unevalses denies rather than allows;PHASE_TOOL_RESULT fails open since the tool already ran by then (policies/types.py:56-59).    
  (c) Message/steering flow during a session — grounded in docs/QUEUE_STEER_DESIGN.md           
  1. User types a follow-up while the agent is busy → held client-side in a pre-POST draft queue (localStorage-persisted, keyed by e server yet(QUEUE_STEER_DESIGN.md:23-36). This is a stated redesign away from the old "POST immediately"
  behavior.
  2. Two ways a queued message reaches the server: (i) auto-flush on idle — when sessionStatus
  goes idle, the FIFO head is sent aii) Steer — user explicitly jumpsthe queue, POSTs immediately regardless of busy state (QUEUE_STEER_DESIGN.md:40-53, 68-76).
  3. Delivery mechanism is harness-sTEER_DESIGN.md:78-87): SDKharnesses (claude-sdk/codex-sdk/pi-sdk) use the runner's live injection gate
  (_live_response_id) via Executor.e(executor.py:573-575, defaultFalse/no-op, overridden per-executor) for deterministic mid-turn delivery. Native harnesses
  without a live-steer RPC (cursor-no send-keys paste into the livetmux pane or queue to a resident extension (supports_enqueue=True); opencode-native has no
  live-steer endpoint at all — a steas a brand-new prompt, promotedonly at the next turn boundary.
  4. Steer is explicitly NOT interrunever cancelled; the message foldsin at the agent's next natural breakpoint (QUEUE_STEER_DESIGN.md:106-112). A separate
  Interrupt button calls turn.cancel
  5. All native run_turn implementations return immediately after delivering the input
  (decoupled from the response) — "ned for native steer"(QUEUE_STEER_DESIGN.md:98-104).
  6. UNCERTAINTY: I did not trace thn-out from runner→server→client(sync vs async at each hop) — that requires reading omnigent/runner/routing.py,
  omnigent/runtime/session_stream.pyession_stream.py, and the server's_runner_transport.py/_runner_ws_tunnel.py, none of which I opened. Given
  server-mapper/host-runner-mapper'se this; recommend merging withtheir trace rather than me re-deriving it.

  (e) Failure/retry — I confirmed only the host-tunnel reconnect (see workflow (a) step 4) and
  runner-exit reporting:

  - HostRunnerExitedFrame (frames.pyport (no result frame) — the hostdaemon watches every runner subprocess it spawned (_watch_runner, connect.py:1192) and on
  unexpected exit composes exit coderver stashes via RunnerExitReports(host_registry.py:39-60, TTL 600s / max 1024 entries) so a client polling a never-connecting
  runner learns why instead of timin
  - If the host tunnel itself is down when a runner dies, the exit report is parked in
  self._unreported_exits (connect.pyand flushed on the next successfulreconnect (_serve_frames, connect.py:1864-1866) — state survives the host-tunnel drop; nothing
  is re-derived, it's just queued cl
  - On server restart: _on_runner_connect callback (server/app.py, ~2230+) re-derives
  session↔runner bindings by queryinconversation_store.list_conversations_by_runner_id(runner_id) (a DB query, not in-memory
  state) for every reconnecting runn session-init for each boundconversation that has an agent_id, re-establishes the SSE relay (_ensure_runner_relay), and
  clears any runner_disconnected sta_publish_runner_recovered_status(require_disconnect_code=True) — so the DB conversation row
  (runner_id, agent_id, labels) is turvives a server restart; thein-memory HostRegistry/relay state is fully re-derived from it.
  - I did NOT trace workflow (b) (seweb/phone → server →host.launch_runner → runner spawn → harness subprocess → first round-trip) in step-by-step
  detail beyond confirming the pieced_text enqueues theHostLaunchRunnerFrame; HostProcess._handle_launch (connect.py:1029-1155) validates harness
  availability (harness_is_configuregent.runner._entry viasubprocess.Popen with binding_token/workspace in env, returns HostLaunchRunnerResultFrame. The
  server-side orchestration that decunch_runner (likely inomnigent/server/routes/sessions.py, not opened) and the runner's own startup sequence in
  runner/_entry.py (create_app, _runot step throughfunction-by-function — recommend the host-runner-mapper's trace be treated as authoritative
  for (b).

  KEY UNCERTAINTIES / GAPS

  1. No confirmed single Harness bas/executor.py) is the confirmedabstract adapter; a Harness concept may exist only in designs/harness-plugin-interface.md
  (unread) as a design-level contracon base class. Check that doc + one *_harness.py file directly.
  2. designs/RUNNER.md and docs/DAEMby section number in code comments(server/app.py, runner/policy.py) but do not exist in the current tree — either renamed/moved
  or deleted; searched the full repo
  3. I did not open omnigent/runtime/policies/engine.py (the server-side PolicyEngine —
  composition/orchestration logic) o/validator.py — my policy/specpicture is from the type/dataclass layer only, not the engine that drives them.
  4. omnigent/runner/app.py is ~7000arness free functions(_auto_create_<harness>_terminal) rather than an obvious class hierarchy — worth confirming
  with host-runner-mapper whether thass I'm missing, or whether it'sgenuinely function-and-module-scoped.
  5. Sequence-diagram-ready sync/asyssage-streaming hop (runner→serverSSE→client) are ungrounded on my end — see gap noted in workflow (c) step 6.

  Files most load-bearing for the diagram: omnigent/spec/types.py,
  omnigent/policies/{base,types,func/policy.py,omnigent/inner/executor.py, omnigent/entities/{conversation,agent}.py, omnigent/tools/mcp.py,
  omnigent/runner/mcp_manager.py, omity,connect}.py,omnigent/server/host_registry.py, docs/QUEUE_STEER_DESIGN.md.

First report is in — the class map and workflow traces are excellent, with policy enforcement,
approval flow, and failure/recoverspecific lines (including agenuinely interesting finding: the server's in-memory state is fully re-derived from the
conversation DB on restart, which tem of record" question). Threeresearchers still working — I'll synthesize once they all report.


⏺ Teammate @class-workflow-mapper fi

Still waiting on the other three rrunner, tmux/native) — nothing newto report yet.


@ host-runner-mapper❯ HOST + RUNNER
  omnigent HOST + RUNNER Architecture Report

  No designs/RUNNER.md exists in this checkout. designs/ has other docs (CUJ-ANALYSIS,
  OBSERVABILITY, SANDBOX_CREDENTIAL_ess-plugin-interface, etc.) butnothing host/runner-specific. Docstrings in git_worktree.py:6 and frames.py:237,360,405 cite
  designs/SESSION_GIT_WORKTREE.md / _SELECTION.md, but neither existsin this checkout — likely relocated or removed. The protocol is documented only inline in
  frames.py/connect.py docstrings.

  1. Module Inventory

  Host (omnigent/host/):

  ┌──────────────────┬─────────────┬────────────────────────────────┐
  │       Path       │    Size     │                     Responsibility                      │
  ├──────────────────┼─────────────┼────────────────────────────────┤
  │                  │             │ HostProcess — main WS loop, reconnect/backoff,          │
  │ connect.py       │ 90KB/2014L  │bprocesses, worktree ops,       │
  │                  │             │ orphan reaper                                           │
  ├──────────────────┼─────────────┼────────────────────────────────┤
  │ frames.py        │ 45KB/1263L  │ 18-kind HostFrameKind enum + 20 frame dataclasses,      │
  │                  │             │                                │
  ├──────────────────┼─────────────┼─────────────────────────────────────────────────────────┤
  │ identity.py      │ 3.9KB/108L  │e from ~/.omnigent/config.yaml  │
  │                  │             │ or managed-sandbox env vars                             │
  ├──────────────────┼─────────────┼────────────────────────────────┤
  │                  │             │ CLI-side helpers hitting server's                       │
  │ daemon_launch.py │ 10.6KB/251L │EST endpoint (launch-or-reuse,  │
  │                  │             │ retries)                                                │
  ├──────────────────┼─────────────┼────────────────────────────────┤
  │ _daemon_entry.py │ 2.1KB/68L   │ python -m omnigent.host._daemon_entry process entry     │
  │                  │             │                                │
  ├──────────────────┼─────────────┼─────────────────────────────────────────────────────────┤
  │ local_server.py  │ 38KB/906L   │cal omnigent server subprocess  │
  │                  │             │ on loopback for local mode                              │
  ├──────────────────┼─────────────┼────────────────────────────────┤
  │ git_worktree.py  │ 18.7KB/463L │ git subprocess wrapper: validate/create/list/remove     │
  │                  │             │                                │
  └──────────────────┴─────────────┴─────────────────────────────────────────────────────────┘

  Runner (omnigent/runner/):

  ┌───────────────────────────┬──────────────┬───────────────────────────────────────────────┐
  │           Path            │      Responsibility                 │
  ├───────────────────────────┼──────────────┼───────────────────────────────────────────────┤
  │                           │     reate_runner_app, app.py:7880)  │
  │ app.py                    │ 883KB/19178L │ — session/terminal/filesystem/mcp routes,     │
  │                           │     to-terminal-create              │
  ├───────────────────────────┼──────────────┼───────────────────────────────────────────────┤
  │ _entry.py                 │ 53KBess entry: env wiring, spec     │
  │                           │              │ resolver, boots WS tunnel, main()             │
  ├───────────────────────────┼─────────────────────────────────────┤
  │ tool_dispatch.py          │ 260KB/6403L  │ execute_tool/dispatch_tool_locally — central  │
  │                           │     utor, sub-agent spawn           │
  ├───────────────────────────┼──────────────┼───────────────────────────────────────────────┤
  │ mcp_manager.py            │ 32KBer — direct stdio MCP pool,     │
  │                           │              │ no-AP/test mode                               │
  ├───────────────────────────┼─────────────────────────────────────┤
  │ proxy_mcp_manager.py      │ 14KB/327L    │ ProxyMcpManager — proxies MCP calls through   │
  │                           │     e                               │
  ├───────────────────────────┼──────────────┼───────────────────────────────────────────────┤
  │ pending_approvals.py      │ 9KB/.Future registry awaiting ASK   │
  │                           │              │ verdicts                                      │
  ├───────────────────────────┼─────────────────────────────────────┤
  │ policy.py                 │ 13KB/336L    │ RunnerToolPolicyGate — local fast-path policy │
  │                           │                                     │
  ├───────────────────────────┼──────────────┼───────────────────────────────────────────────┤
  │ resource_registry.py      │ 57KBeRegistry —                     │
  │                           │              │ terminal/environment lifecycle                │
  ├───────────────────────────┼─────────────────────────────────────┤
  │ environment_filesystem.py │ 32KB/844L    │ CallerProcessFilesystem — typed FS adapter    │
  │                           │      OSEnvironment                  │
  ├───────────────────────────┼──────────────┼───────────────────────────────────────────────┤
  │                           │      server-side                    │
  │ routing.py                │ 10KB/280L    │ conversation→runner dispatch code that lives  │
  │                           │     e (imports                      │
  │                           │              │ ConversationStore/TunnelRegistry)             │
  ├───────────────────────────┼─────────────────────────────────────┤
  │ identity.py               │ 6KB/142L     │ Runner identity, tunnel binding-token         │
  │                           │     ret-stripping                   │
  ├───────────────────────────┼──────────────┼───────────────────────────────────────────────┤
  │ uc_function.py            │ 9.5Kricks Unity Catalog SQL         │
  │                           │              │ functions                                     │
  └───────────────────────────┴─────────────────────────────────────┘

  Transports (omnigent/runner/transp
  __init__.py documents 3 phases: UDS (uds.py, legacy/test), TCP (tcp.py, legacy/test), WS
  tunnel (ws_tunnel/, production). wdataclasses), limits.py(timing/size constants), registry.py (server-side runner_id→WS map), serve.py (runner-side WS
  client loop), transport.py (WSTunnseTransport), server-side).

  Sandboxing: cli_sandbox.py is a fasandbox {create,auth,connect}, CLIfor provisioning a remote cloud sandbox (Modal/Daytona/Lakebox/Islo) to host an Omnigent host
  process, unrelated to OS process sing lives in inner/sandbox.py(policy/registry), inner/bwrap_sandbox.py (Linux), inner/seatbelt_sandbox.py (macOS),
  re-exported via sandbox/bwrap.py/s

  2. Host Frame Protocol Table (host

  All frames JSON dataclasses under s.py:37):

  ┌────────────────────────────────┬───────┬───────────────────────┐
  │             Frame              │  Direction  │     Purpose      │      Key fields       │
  ├────────────────────────────────┼───────┼───────────────────────┤
  │                                │             │ Register host on │ version, name,        │
  │ HostHelloFrame (:64)           │nect   │ runners[],            │
  │                                │             │                  │ configured_harnesses  │
  ├────────────────────────────────┼───────┼───────────────────────┤
  │                                │             │                  │ request_id,           │
  │ HostLaunchRunnerFrame (:93)    │nner   │ binding_token,        │
  │                                │             │                  │ workspace,            │
  │                                │       │ session_id, harness   │
  ├────────────────────────────────┼─────────────┼──────────────────┼───────────────────────┤
  │ HostLaunchRunnerResultFrame    │       │ request_id, status,   │
  │ (:123)                         │ host→server │ Launch outcome   │ runner_id, error,     │
  │                                │       │ error_code            │
  ├────────────────────────────────┼─────────────┼──────────────────┼───────────────────────┤
  │ HostStopRunnerFrame (:150)     │a      │ request_id, runner_id │
  │                                │             │ runner           │                       │
  ├────────────────────────────────┼───────┼───────────────────────┤
  │ HostStopRunnerResultFrame      │ host→server │ Stop outcome     │ request_id, status,   │
  │ (:164)                         │       │ error                 │
  ├────────────────────────────────┼─────────────┼──────────────────┼───────────────────────┤
  │                                │       │                       │
  │ HostRunnerExitedFrame (:180)   │ host→server │ unexpected-death │ runner_id, error      │
  │                                │       │                       │
  ├────────────────────────────────┼─────────────┼──────────────────┼───────────────────────┤
  │ HostStatFrame/Result           │       │ path / exists, type,  │
  │ (:204/227)                     │  / reply    │ filesystem path  │ canonical_path        │
  ├────────────────────────────────┼───────┼───────────────────────┤
  │ HostListDirFrame/Result        │ server→host │                  │ path, limit, after,   │
  │ (:297/328)                     │browse │ before / entries[],   │
  │                                │             │                  │ has_more              │
  ├────────────────────────────────┼───────┼───────────────────────┤
  │                                │             │                  │ repo_path,            │
  │ HostCreateWorktreeFrame/Result │       │ branch_name,          │
  │  (:357/377)                    │  / reply    │ worktree         │ base_branch /         │
  │                                │       │ worktree_path, branch │
  ├────────────────────────────────┼─────────────┼──────────────────┼───────────────────────┤
  │ HostRemoveWorktreeFrame/Result │ktree  │ worktree_path,        │
  │  (:401/425)                    │  / reply    │                  │ branch, delete_branch │
  ├────────────────────────────────┼───────┼───────────────────────┤
  │ HostListWorktreesFrame/Result  │ server→host │ List worktrees   │ repo_path /           │
  │ (:441/459)                     │       │ worktrees[]           │
  ├────────────────────────────────┼─────────────┼──────────────────┼───────────────────────┤
  │ HostCreateDirFrame/Result      │ost    │ path                  │
  │ (:479/498)                     │  / reply    │                  │                       │
  └────────────────────────────────┴───────┴───────────────────────┘

  Plus app-level PingFrame/PongFrame39) for tunnel keepalive on bothhost and runner tunnels.

  3. Runner Internal Architecture — Data Flow

  Harness subprocess → app.py's /v1/sessions/{id}/mcp/execute (app.py:18151), or a harness's
  policy_evaluation.requested SSE evte_policy_via_omnigent(app.py:6430) POSTs to server /policies/evaluate, relays verdict back as policy_verdict. For
  runner-owned categories, tool_dispispatch.py:4389) either delegatesMCP calls to the manager (mcp_manager.call_tool) or dispatches sys_os_*/REST/terminal/subagent
  tools locally.

  Two enforcement points:
  - Local fast path: RunnerToolPolicyGate._evaluate_policies (policy.py:228) — only used with
  RunnerMcpManager (no-AP/test mode)K escalates viapending_approvals.wait_for_user_approval (pending_approvals.py:133), a Future keyed by
  elicitation_id, resolved when the ons/{id}/events.
  - Server round-trip (AP mode, ProxyMcpManager): _evaluate_policy_via_omnigent (app.py:6430) —
  TOOL_CALL fails closed on error, LOOL_RESULT fail open(app.py:6449-6459); ASK parks server-side, runner awaits via pending_approvals with a 1-day
  timeout (app.py:6497-6505, pending

  mcp_manager.py vs proxy_mcp_managexy_mcp_manager.py:14-22):RunnerMcpManager (no-AP/test) owns direct stdio MCP subprocess connections (LRU pool, capacity
  8, mcp_manager.py:44) and enforcesManager (AP mode, production)holds no connections — every call_tool (:178) POSTs JSON-RPC to the server's
  /v1/sessions/{id}/mcp, which owns y and enforces policy there. Bothexpose the same interface, interchangeable at dispatch sites.

  Harness spawning: POST /v1/sessions (app.py:8767) triggers per-harness
  _auto_create_{claude,codex,cursor,ions (e.g. app.py:5347 for Claude), which call resource_registry.launch_required_terminal/launch_auxiliary_terminal
  (resource_registry.py:758,791) → _cycle (:824). The actualsubprocess/tmux mechanics are outside runner/, owned by HarnessProcessManager
  (omnigent.runtime.harnesses.procesntry.py:865) — not directlysubprocess.Popen inside app.py.

  Agent spec resolution: _entry.py:849 create_app() builds a spec_resolver (:952) wrapping
  _resolve_agent_spec_from_server (:ssions/{id}/agent/contents, checksX-Agent-Session-Scoped header (fail-safe: unknown → no ${VAR} expansion, :823-826), caches the
  extracted bundle per runner, calls→ ResolvedSpec (app.py:6622).

  4. Transport Architecture — Defini

  Each runner opens its own dedicatehe server; it is NOT tunneledthrough the host's WS connection. Two independent WS clients:
  - Runner→server: serve.py:547 webs,...), tunnel_url from_tunnel_url(server_url, runner_id) (serve.py:914), hitting server route
  @router.websocket("/runners/{runneutes/runner_tunnel.py:344).
  - Host→server: separate connection in host/connect.py against server/routes/host_tunnel.py.

  The server does multiplex many server-side httpx requests onto one runner's WS via
  WSTunnelTransport (transport.py:10stry (registry.py:1-9), but that WS belongs solely to the runner (dialed by the runner itself). The host only launches the
  subprocess and injects env (host/ctches/reports its exit — it doesnot proxy runner traffic.

  RUNNER_TUNNEL_BINDING_TOKEN_ENV_VAR = "OMNIGENT_RUNNER_TUNNEL_BINDING_TOKEN"
  (runner/identity.py:21). Set at hounch, injected into the runnersubprocess env) and cli.py:2804 (local omnigent run, secrets.token_urlsafe(32) fallback). Read
  at runner/_entry.py:580-594. Sent igent-Runner-Tunnel-Token on theWS handshake (serve.py:546). Purpose is dual: auth and session binding —
  token_bound_runner_id(token) (idenashes the token into adeterministic runner_id; the server route's path param must match, cryptographically
  preventing one caller from claimina shared server. Explicitlystripped before any spawned child process inherits it (RUNNER_AUTH_SECRET_ENV_VARS,
  identity.py:47-55).

  Ping/Pong: library-level keepalivetimeout=90s (serve.py:555-556,ws_tunnel/limits.py:24-25) plus app-level PingFrame/PongFrame JSON frames sent every 30s by
  the server, echoed by the runner (sed pongs (limits.py:9-10 comment)declares the peer dead at ~90s.

  5. Reconnection / Retry / Backoff (host)                                                      
  Constants (host/connect.py:248-250): _RECONNECT_BASE_S=0.5, _RECONNECT_CAP_S=10.0,            _RECONNECT_JITTER=0.5. HostProcessnect_and_serve() (:1726); backoffresets to base on success (:1635). Fatal HostConnectError (auth/authorization/outdated server)re-raises without retry → SystemEx failures: "recycle" signals(close code 1012/1001, "going away", or 502/no-close-frame on non-loopback servers) get an    immediate base-delay retry; otherwbackoff =min(backoff*2*(1+random()*jitter), cap) (:1673-1697). Loopback servers skip the fast-recycle  path deliberately to avoid re-regi1672). Login-redirect fatalthreshold: _LOGIN_REDIRECT_FATAL_ATTEMPTS=3 (:480), but only for hosts that have never        connected — an already-connected hever.
                                                                                                6. Lifecycle / Failure Narrative
                                                                                                Runner launch: _handle_launch (conorkspace, builds env via_build_runner_env (allowlist-based, :261,499), subprocess.Popen([sys.executable, "-m",        "omnigent.runner._entry"], ...) (:pawns _watch_runner (:1192) polling proc.poll() every _RUNNER_WATCH_INTERVAL_S=0.5s (:135).                                       
  Crash path: unexpected exit while still tracked → composes exit code + log tail →             _report_runner_exit (:1220-1244) eame and sends over the host's liveWS. If send fails (tunnel down), it's parked in self._unreported_exits[runner_id] (:1244) and flushed on next successful reconneide, host_tunnel.py:452-464 matches the frame, logs it, calls runner_exit_reports.record(...) — surfacing "offline, and here's    why" to status pollers rather thanatic respawn/session-resume istriggered by this frame — it's purely diagnostic. Session resumption instead lives client-sidein daemon_launch.launch_or_reuse_dch.py:166-229), retrying transient 409 "host offline" across a ~16.5s budget (_RETRY_DELAYS_S=(0.5,1.0,1.5,2.0,2.5,3.0,3.0,3.0), :209).
                                                                                                Also present: an orphan reaper (_ot.py:691) sweeping every 2.0s(_ORPHAN_REAP_INTERVAL_S, :144) to reap grandchild tool processes (node/npm/chromium/tmux)    left by a dead runner.
                                                                                                7. Host Identity & Daemon/Local Mo
                                                                                                HostIdentity(host_id, name) (ident"host_" + uuid4().hex.Loaded/created from ~/.omnigent/config.yaml's host: section, or overridden entirely (no disk  write) by OMNIGENT_HOST_ID+OMNIGENmanaged sandboxes(identity.py:28-29,77-85). OMNIGENT_HOST_TOKEN carries tunnel auth, sent as                   X-Omnigent-Host-Token.
                                                                                                Daemon entry (_daemon_entry.py:20-cal (mutually required), callsrun_host_process(server_url=...) — same top-level function used by omnigent host directly.    Local mode (local_server.py) does ightweight server — itspawns/manages the lifecycle of the same full omnigent server process bound to loopback       (_spawn_local_server, :584-666, poith a pidfile+config-signaturecheck to reuse a healthy instance (:442,90). Local mode is architecturally identical to remotefrom the host daemon's perspectiveting the server.
                                                                                                8. Git Worktrees (host/git_worktre
                                                                                                All server-initiated via the frameheduling), dispatched atconnect.py:1952-1957. Naming: <repo-parent>/<repo-name>-worktrees/<sanitized-branch>          (slashes→-), collision suffix -2..1,20-21). Creation resolves themain worktree first via git worktree list --porcelain so new worktrees are always siblings notnested (:373-379), rejects existinns git worktree add -b <branch><path> --end-of-options <base> guarding against flag injection (:392-399). Removal resolves   the main repo via git rev-parse --worktree remove --force, thenoptionally git branch -D in that order since git refuses branch deletion while checked out    (:429-462). All git calls use argvs timeout (:16-18).
                                                                                                9. Sandboxing
                                                                                                Wraps agent-spawned OS subprocessest/runner processes themselves.Shared interface inner/sandbox.py: SandboxPolicy (read/write roots, allow_network,            dotfile-hiding, egress relay sockected via agent YAMLos_env.sandbox.type or platform default (_default_sandbox_for_platform, :851).                - Linux bubblewrap (inner/bwrap_same="linux_bwrap"): requires bwrapon PATH or fails loud; cwd read-only unless write_paths:["."]; unshares network namespace     unless allow_network.
  - macOS seatbelt (inner/seatbelt_sandbox.py:1-40,329-343, type_name="darwin_seatbelt"):       prepends sandbox-exec -f <profile-tten to mode-0600 tempfile (not -pinline) to hide internals from ps aux; omits mach-priv-host-port/iokit-open; narrows /dev to  read-only plus literal allows for tracehelper (:18-28).
  Both registered at import time; callers include inner/terminal.py:788,898,                    inner/acp_executor.py:235, tools/b137.
                                                                                                10. Key Class Relationships
                                                                                                - HostProcess (connect.py) owns: W_exits, spawned Popen handles,_watch_runner/orphan-reaper tasks. Creates: HostLaunchRunnerResultFrame, HostRunnerExitedFrameinstances.
  - SessionResourceRegistry (runner) owns TerminalRegistry; depends on external                 HarnessProcessManager for actual p
  - RunnerMcpManager/ProxyMcpManager — interchangeable, created once per runner or per-session; consumed by tool_dispatch.execute_e constructed per spec_hash, cached on RunnerMcpManager.                                                                          - pending_approvals module-global of both local-policy ASK escalation and proxy-mode MRTR ASK.                                                                      - RunnerRouter (routing.py) — desp server-side code (ownsTunnelRegistry+ConversationStore refs), not instantiated in the runner process.               - TunnelRegistry/WSTunnelTransportr_id→WS, frame server-originatedhttpx calls over it.                                                                          
  Uncertainties (explicit)                                                                      
  - designs/SESSION_GIT_WORKTREE.md / designs/SESSION_WORKSPACE_SELECTION.md cited in code but  absent from this checkout.
  - Exact point where a tmux pane/OS process is forked for a harness lives in                   omnigent.terminals.TerminalRegistroutside runner/ — only the callboundary was confirmed, not internals.                                                        - RunnerRouter's presence in the ring server-side code is unexplained — possibly a packaging artifact.                                                              - Did not fully read tool_dispatchKB) — onlyexecute_tool/dispatch_tool_locally/subagent paths verified; most _execute_*_tool              REST/timer/browser handlers identi
  - _ping_loop/"3 missed pongs" logic inferred from a comment in ws_tunnel/limits.py:9-10; the  actual loop body in runner_tunnel.read to confirm verbatim.
  - Did not verify whether HostRunnerExitedFrame ever triggers automatic respawn elsewhere in   server code outside host_tunnel.py
  - Full bwrap seccomp profile / SBPL rule set beyond cited excerpts unverified (large files,   sampled not exhaustively read).
  - TCP/UDS transports marked "Phase 2/3" in transports/__init__.py:3-5; unclear if any         production path still selects themsole production transport.
                                                                                                All findings from three parallel rts against the live checkout at/Users/kielay/Delta-Analytical-Corp/delta-data-platform/apps/omnigent. No files were modified.
Host+runner report is in — and it settles a key architectural question: each runner opens its own WebSocket to the server (the htcher, not a traffic proxy), with a binding token that cryptographically derives the runner's identity. Two reports still pending (server, tmux/native wrappers).
                                                                                                
@ server-mapper❯ Omnigent server subsystem report ready                                         Omnigent Server Subsystem — Resear
                                                                                                Module inventory
                                                                                                Path: omnigent/cli.py:2916 (omni sup)
  Responsibility: Real process entrypoint; builds app via create_app() and runs uvicorn         ──────────────────────────────────
  Path: omnigent/__main__.py:3                                                                  Responsibility: python -m omnigent
  ────────────────────────────────────────                                                      Path: omnigent/host/local_server.p
  Responsibility: Not a second server. Pidfile-tracked supervisor that spawns sys.executable -m omnigent.cli server ... as a subprlt port 6767) so repeated local
  CLI invocations reuse one instance. The spawned process is the identical FastAPI server       described below — locally, host/rul talk to it over the same WS
  tunnel protocols as a remote deployment                                                       ──────────────────────────────────
  Path: omnigent/server/app.py                                                                  Responsibility: FastAPI app factor63), lifespan (app.py:1231), router                                                                                              mounting, middleware
  ────────────────────────────────────────                                                      Path: omnigent/server/routes/*.py
  Responsibility: REST/WS route handlers (table below)                                          ──────────────────────────────────
  Path: omnigent/server/auth.py, oidc.py, oidc_access.py, accounts_*.py, passwords.py           Responsibility: Auth (see Auth sec
  ────────────────────────────────────────                                                      Path: omnigent/server/host_registrol.py, smart_routing.py,
  managed_hosts.py, performance_metrics.py, admin_list.py, bundles.py                           Responsibility: In-memory registriP proxy pool, model routing, admin
  roster, metrics                                                                               ──────────────────────────────────
  Path: omnigent/host/connect.py, identity.py, frames.py, daemon_launch.py                      Responsibility: Host daemon: dialss/{id}/tunnel, spawns runner
  subprocesses                                                                                  ──────────────────────────────────
  Path: omnigent/runner/transports/ws_tunnel/*                                                  Responsibility: Runner-side WS tunmes.py, registry.py, transport.py)
  ────────────────────────────────────────                                                      Path: omnigent/db/db_models.py, mi
  Responsibility: SQLAlchemy models + Alembic migrations                                        ──────────────────────────────────
  Path: omnigent/stores/*                                                                       Responsibility: Store layer (reposB models
  ────────────────────────────────────────                                                      Path: omnigent/entities/*
  Responsibility: Plain dataclasses/Pydantic domain objects (Conversation, Agent, Permission,   Policy, etc.)
                                                                                                Framework: FastAPI on Starlette (s= FastAPI(title="Omnigent Server",lifespan=_lifespan) at app.py:1383. No CORSMiddleware found anywhere (grepped) — likely       same-origin via the bundled SPA sted by a deployment-side proxy;unconfirmed. Middleware present: WS-metrics counter, WebSocketOriginMiddleware (CSWSH guard), request-metrics, gzip on the stati

  Route inventory (selected)

  sessions.py is the dominant file (ts under /v1/sessions). Highlights:

  ┌─────────────────────────────┬──────────┬──────────────────────┐
  │            Path             │   Method(s)   │     file:line     │       Purpose        │
  ├─────────────────────────────┼──────────┼──────────────────────┤
  │ /v1/sessions                │ POST, GET     │ sessions.py:14287 │ create/list sessions │
  │                             │          │                      │
  ├─────────────────────────────┼───────────────┼───────────────────┼──────────────────────┤
  │                             │ GE:14744 │ session CRUD (incl.  │
  │ /v1/sessions/{id}           │ ETE           │ ,15429,20435      │ include_items        │
  │                             │          │ snapshot)            │
  ├─────────────────────────────┼───────────────┼───────────────────┼──────────────────────┤
  │                             │          │ live session-list    │
  │ /v1/sessions/updates        │ WebSocket     │ sessions.py:15164 │ push, replaces 4s    │
  │                             │          │ HTTP poll            │
  ├─────────────────────────────┼───────────────┼───────────────────┼──────────────────────┤
  │                             │ GE       │ live-tail of turn    │
  │ /v1/sessions/{id}/stream    │ text/event-st │ sessions.py:20245 │ events for one       │
  │                             │ re       │ session              │
  ├─────────────────────────────┼───────────────┼───────────────────┼──────────────────────┤
  │                             │          │ submit input/interru │
  │ /v1/sessions/{id}/events    │ POST          │ sessions.py:19157 │ pt/approval into a   │
  │                             │          │ session              │
  ├─────────────────────────────┼───────────────┼───────────────────┼──────────────────────┤
  │ /v1/sessions/{id}/resources │ GE:17399 │ env filesystem,      │
  │ [...]                       │ PATCH/DELETE  │ -18930            │ terminals, files,    │
  │                             │          │ browser actions      │
  ├─────────────────────────────┼───────────────┼───────────────────┼──────────────────────┤
  │ /v1/hosts, /v1/hosts/{id}   │ GE1,371  │ list/get registered  │
  │                             │               │                   │ hosts                │
  ├─────────────────────────────┼──────────┼──────────────────────┤
  │ /v1/hosts/{id}/runners      │ POST          │ hosts.py:408      │ launch a runner on a │
  │                             │          │  connected host      │
  ├─────────────────────────────┼───────────────┼───────────────────┼──────────────────────┤
  │ /v1/hosts/{id}/tunnel       │ We.py:12 │ host daemon          │
  │                             │               │ 1                 │ control-plane tunnel │
  ├─────────────────────────────┼──────────┼──────────────────────┤
  │ /v1/runners/{id}/tunnel     │ WebSocket     │ runner_tunnel.py: │ runner HTTP-over-WS  │
  │                             │          │ tunnel               │
  ├─────────────────────────────┼───────────────┼───────────────────┼──────────────────────┤
  │ /v1/runners/{id}/token      │ POel.py: │ mint runner binding  │
  │                             │               │ 284               │ token                │
  ├─────────────────────────────┼──────────┼──────────────────────┤
  │ /v1/sessions/{id}/resources │ WebSocket     │ terminal_attach.p │ browser xterm.js ↔   │
  │ /terminals/{tid}/attach     │          │ runner PTY bridge    │
  ├─────────────────────────────┼───────────────┼───────────────────┼──────────────────────┤
  │                             │   .py:13 │ login/callback/invit │
  │ /auth/* (OIDC)              │ GET/POST      │ 1-558             │ e/logout/CLI         │
  │                             │          │ device-code          │
  ├─────────────────────────────┼───────────────┼───────────────────┼──────────────────────┤
  │                             │          │ password             │
  │ /auth/* (accounts)          │ GET/POST/DELE │ routes/accounts_a │ login/register/setup │
  │                             │ TE739    │ /magic-link/user     │
  │                             │               │                   │ admin                │
  └─────────────────────────────┴──────────┴──────────────────────┘

  Every non-SSE, non-WS content endpalidated by ~61 Pydantic models inserver/schemas.py (4066 lines).

  Data model summary

  Two SQLAlchemy DeclarativeBases, OmnigentBase and ConversationBase (db_models.py:38-58), can
  point at separate physical DBs. Ev workspace_id (tenant partition,added by migration r1a2b3c4d5e6).

  Table: agents
  Key fields: bundle_location (artif kind, description (compressed)
  Durable vs derived: Durable metadata; blob external
  file:line: db_models.py:112-169
  ────────────────────────────────────────
  Table: files
  Key fields: filename, bytes, content_type, session_id
  Durable vs derived: Durable metadatStore (local/S3/Databricksvolumes)
  file:line: db_models.py:172-214
  ────────────────────────────────────────
  Table: users
  Key fields: is_admin, password_hash, last_login_at
  Durable vs derived: Durable
  file:line: db_models.py:217-256
  ──────────────────────────────────
  Table: account_tokens
  Key fields: kind (invite/magic), ures_at
  Durable vs derived: Durable, short-TTL single-use
  file:line: db_models.py:259-317
  ────────────────────────────────────────
  Table: session_permissions
  Key fields: (user_id, conversation_id) → level (1=read/2=edit/3=manage)
  Durable vs derived: Durable ACL
  file:line: db_models.py:320-371
  ──────────────────────────────────
  Table: omnigent_conversation_metadata
  Key fields: runner_id, host_id, se (compressed JSON)
  Durable vs derived: Persisted-as-source-of-truth runtime snapshot
  file:line: db_models.py:374-422
  ────────────────────────────────────────
  Table: agent_configuration
  Key fields: agent_id, model_override, harness_override
  Durable vs derived: Durable
  file:line: db_models.py:425-480
  ──────────────────────────────────
  Table: conversations
  Key fields: title, parent_conversan_id, next_position (monotonic
  allocator)
  Durable vs derived: Durable
  file:line: db_models.py:483-561
  ──────────────────────────────────
  Table: conversation_items
  Key fields: position (unique per c, type/status
  Durable vs derived: Durable, append-only — no update path found for item bodies
  file:line: db_models.py:564-647
  ────────────────────────────────────────
  Table: conversation_items_fts
  Key fields: FTS5 virtual table (SQLite/Cloudflare D1 only)
  Durable vs derived: Fully derived
  file:line: db/utils.py:708-748
  ──────────────────────────────────
  Table: comments, policies, hosts, user_daily_cost
  Key fields: —
  Durable vs derived: hosts.status/updated_at doubles as heartbeat (semi-derived);
  user_daily_cost is an explicit O(1
  file:line: db_models.py:705-1051

  Engines: SQLite (default, sqlite:///omnigent.db, WAL+PRAGMAs, db/utils.py:231-286), PostgreSQL
  via psycopg (db/utils.py:179-286),tgres + OAuth token refresh,db/utils.py:35-173), MySQL via pymysql, and Cloudflare D1 (SQLite-over-HTTP). All access is
  synchronous SQLAlchemy run via asy DB drivers.

  Compression: CompressedText type tses 6 "opaque" columns neverqueried in SQL (agents.description, session_state/session_usage/terminal_launch_args,
  comments.body/anchor_content) — db

  Notable migration: p1a2b3c4d5e6_re remaining DB-enforced foreign keys per internal "Rule R032" — the application layer is now solely responsible for cascading
  deletes (confirmed by hand-written:736-757).

  Blob storage: ArtifactStore (localniform put/get/delete/existsinterface) holds all file/bundle bytes; the SQL DB stores only pointer keys — confirmed no
  blob columns exist for agents.bund

  Session/conversation model & clien

  "Session" (API/product term) and "onversation.py:26, DB conversations table) are the same entity. Key fields: identity/hierarchy (root_conversation_id,
  parent_conversation_id, kind), ageplacement, workspace (immutablerealpath), session_state/session_usage (mutable JSON), and per-session overrides
  (model_override, harness_override,ation.py:191-216.

  Messages/tool-calls/reasoning are versation_items rows ordered by amonotonic position allocated under a row lock (sqlalchemy_store.py:1770-1849), not simple
  timestamps. designs/OBSERVABILITY. lower-level wire-format "durableevent log" is a future, not-yet-built Phase-3 item — conversation_items is today's closest
  equivalent, already durable.

  Protocol summary:

  Mechanism: GET /v1/sessions/{id}/s
  Direction: server→client
  Purpose: live turn-event tail (no  snapshot on reconnect)
  file:line: sessions.py:20245
  ──────────────────────────────────
  Mechanism: WS /v1/sessions/updates
  Direction: bidirectional
  Purpose: session-list live diff push
  file:line: sessions.py:15164
  ────────────────────────────────────────
  Mechanism: In-process pub/sub sessibe
  Direction: server-internal
  Purpose: fans SSE out from workfloon asyncio.Queue, no DB, events
  dropped if no subscriber (at-most-once)
  file:line: runtime/session_stream.
  ────────────────────────────────────────
  Mechanism: WS /v1/hosts/{id}/tunne
  Direction: bidirectional
  Purpose: host control-plane (launc
  file:line: host_tunnel.py:121
  ──────────────────────────────────
  Mechanism: WS /v1/runners/{id}/tunnel
  Direction: bidirectional
  Purpose: runner HTTP-over-WS + terminal-channel multiplex
  file:line: runner_tunnel.py:344
  ────────────────────────────────────────
  Mechanism: WS /v1/sessions/{id}/rettach
  Direction: bidirectional
  Purpose: raw PTY bytes, proxied thf remote
  file:line: terminal_attach.py:131
  ──────────────────────────────────
  Mechanism: presence.py in-memory registry
  Direction: server-internal
  Purpose: who's viewing a session tree; 15s reconnect grace window; dies with process, never
  persisted
  file:line: presence.py:1-65

  Clients (web/mobile/desktop, and the sdks/python-client/omnigent_client package) are thin
  projections: the SDK's BlockStream blocks purely from the SSE eventsequence, with no local persistent store found in either SDK package — consistent with
  designs/CUJ-ANALYSIS.md:457 explicon history (source of truth)" asserver-side.

  Auth

  Three mutually-exclusive user-identity modes, selected by OMNIGENT_AUTH_PROVIDER
  (server/auth.py:266-307):
  - header (default) — trusts an upstream-injected identity header (X-Forwarded-Email by
  default); this is the mechanism Daocs/databricks.md:53-55, confirmedvia server/auth.py:617-619) — there is no Databricks-specific OIDC code path, Databricks flows
  through generic trusted-header aut
  - oidc — full authorization-code + PKCE flow, generic-OIDC-discovery or hardcoded GitHub
  provider (server/oidc.py:26-355).
  - accounts — built-in argon2id password auth, first-user-becomes-admin bootstrap that
  self-disables once any admin exist1-24,243-389).

  Both OIDC and accounts share an HS_Host-ap_session) validated inUnifiedAuthProvider._check_cookie (server/auth.py:477-539).

  There is no auth_tokens table — the token table added by the migration named for it is
  actually account_tokens (db_modelsite/magic-link tokens verbatim (not hashed), single-use via atomic redeemed_at IS NULL update.

  Host auth (host_tunnel.py:121-235, before ws.accept()): either (a) a user's Bearer JWT (same
  cookie/session mechanism, for userons), or (b) X-Omnigent-Host-Token— a random launch token minted for server-provisioned managed sandboxes, stored server-side
  only as a SHA-256 digest (hosts.tooken — host_store.py:155-168),scoped to one host_id, implicitly rotated on sandbox relaunch. Runner sub-processes get a
  separate one-time binding_token mient of host auth.

  System of record analysis

  Server is durably authoritative foory(conversations/conversation_items, monotonic-position append log), the host registry's
  persistent fields (hosts table, upat), policies, permissions,comments, users, and agent configs — all backed by the SQL DB.

  Host holds only in-memory state locally (live WS connection object, spawned-runner subprocess
  handles) and syncs up to the serveund tunnel; the server never opensa connection to a host, and never mirrors the host's filesystem — it only pulls on demand via
  request/response frames (host.stat

  Runner holds the actual agent proc workspace files, and tmux/PTYstate entirely locally; these never get replicated to the server — only the resulting
  conversation turns (via the runnerC) get persisted asconversation_items as a side effect of the server handling each response.

  Sync direction: overwhelmingly push/RPC-over-persistent-tunnel, always initiated outbound by
  the host/runner (never server→hostone asynchronous, unprompted pushfrom host to server is host.runner_exited (crash notification) — otherwise every frame is a
  response to a server-initiated reqalready-open tunnel.host/local_server.py does not change this picture: it only supervises a subprocess running the
  identical server code, so even fuletups route host/runner/clienttraffic through the same /v1/hosts/…/tunnel and /v1/runners/…/tunnel WS protocol against a
  loopback instance.

  Uncertainties (consolidated across

  - No CORSMiddleware registration fpp.py — CORS handling location(proxy vs none-needed) is unconfirmed.
  - sessions.py is ~21.8k lines; rouom decorator/path extraction, notfull manual reads of every of the ~60 handler bodies.
  - A /health/subscribe WebSocket roCUJ-ANALYSIS.md:456 but could notbe located by grep in server/ — may be named differently, or the doc may be
  stale/aspirational.
  - The Phase-3 "durable event log" in designs/OBSERVABILITY.md:424 is a proposed future
  addition; unclear if any partial ists outside what was searched.
  - The exact call site that mints a managed-sandbox host's initial launch token was not located
  within the files reviewed.
  - omnigent/runner/transports/ws_tunnel/registry.py (978 lines) and serve.py (948 lines) —
  runner-side reassembly/backpressure not read in full; assumedsymmetric to the host side based on shared constants, not independently confirmed.
  - Store-layer internals for convertore.py (2851+ lines,search/forking/daily-cost logic) and the three artifact_store backends were reviewed only at
  the method-signature level, not fu

  All source citations above are fild by five independent read-onlyresearch passes over the /Users/kielay/Delta-Analytical-Corp/delta-data-platform/apps/omnigent
  codebase; no files were modified.

Server report is in — the "one autjections" question now has a cleananswer: the server's SQL store (append-only conversation_items with monotonic positions) is
the system of record; web/phone/SDtions rebuilt from SSE streams with no local store; hosts and runners keep only local runtime state that syncs up. One report
left — the tmux/native-wrapper deethe page.


⏺ Teammate @host-runner-mapper finished

Still waiting on the tmux/native-wrapper researcher — that's the last report before I
synthesize.


⏺ Teammate @server-mapper finished

One researcher (tmux/native wrappers) still running — holding for their report.


@ tmux-native-mapper❯ tmux + native
  Native Harness Wrappers & tmux — Architecture Report

  All paths relative to /Users/kielay/Delta-Analytical-Corp/delta-data-platform/apps/omnigent.
  Read-only research, nothing modifi

  1. tmux invocation inventory

  Base command / socket. omnigent/in(_tmux_base_cmd): tmux -S<socket_path> -f /dev/null. The -f /dev/null deliberately ignores the user's own ~/.tmux.conf.
  Socket path is not hash-derived: ttempfile.mkdtemp(prefix="omnigent-terminal-")/tmux.sock — every TerminalInstance gets its own
  private tmux server, not a shared .

  Session naming. tmux_target is haring "main" (terminal.py:951-953).Since each terminal owns a private server, there's never a need to disambiguate — the socket
  path is the identity.

  new-session. terminal.py:1123-1147-x 80 -y 24 -c <cwd> <inner_cmd>,preceded by option commands (terminal.py:226-377): history-limit,
  extended-keys/mouse/focus-events,  block (prefix None, unbinding thewhole prefix table + mouse-3 context menu so an attached human can't spawn stray panes), and a
  custom status line. Optionally foln (terminal.py:257-289) — theload-bearing option for "session survives the CLI exiting."

  send-keys. Literal text: send-keys -l -t main <chunk>, chunked at 1024 chars
  (terminal.py:1165-1213) because tmps a single command near 16KB.Special keys (Enter, Escape, Ctrl-A/K/U) sent as separate send-keys -t main <key> calls
  throughout every *_native_bridge.pe_bridge.py:2543-2597,hermes_native_bridge.py:707-740, goose_native_bridge.py:294-328,
  kiro_native_bridge.py:592-755).

  Buffer-paste (bypasses the 16KB carminal.py — duplicated per harness: claude_native_bridge.py:2559-2578 writes payload to a temp file, load-buffer -b
  omnigent-paste <path>, then paste-paste -t <target> (-p =bracketed-paste so embedded newlines don't submit early). Same pattern in
  hermes_native_bridge.py:~716-724, 0-306,antigravity_native_bridge.py:~1048.

  capture-pane. Reading pane content back: terminal.py:1215-1240 (read()), idle-watcher polling
  (terminal.py:1409-1416, 1s interva:153-228 (seeds a freshly-attachedbrowser client, since control-mode clients only see output after attach).

  pipe-pane. Confirmed absent — grep -rn "pipe-pane" omnigent/ returns zero hits anywhere in the
  package. No background pane loggin; live viewing goes through theattach bridges instead.

  kill-session / kill-server / liveness. terminal.py close() uses kill-server (kills the whole private server). Per-harness bridgde_native_bridge.py:2688,hermes_native_bridge.py:833, goose_native_bridge.py:354, kiro_native_bridge.py:653,          qwen_native_bridge.py:789, cursor_codex_native_process_registry.py:412 (orphan reaping). Notably, core terminal.py never calls has-session — it uses list-panes -instead(terminal.py:1634-1656,1706-1766), because remain-on-exit keeps the session alive afterCLI exits, so has-session alone ca" from "the agent crashed."has-session is used, but only in the per-harness bridge readiness probes and the REPL: hermes_native_bridge.py:575, goosetigravity_native_bridge.py:1035,kimi_native_bridge.py:257, cursor_native_bridge.py:550, kiro_native_bridge.py:343,
  codex_native_process_registry.py:3

  Other commands. display-popup for  (native_cost_popup.py);list-clients -t main as the "is a human attached" liveness signal feeding the idle-pane reaper
  (native_cost_popup.py:45,106); attnto a real PTY(terminals/ws_bridge.py:547) or via -C control mode (terminals/control_bridge.py:354-357);
  send-keys -H <hex> for control-moduld corrupt tmux's line-basedcontrol parser); set-option -p -t <pane> in the REPL's optional (disabled-by-default)
  outer-pane integration (repl/_tmuxe managed terminals).

  2. Per-wrapper architecture (claudative case)

  claude_native.py's docstring: "cre session, launches claude throughthe existing runner terminal resource API, then attaches the local TTY to the existing
  terminal WebSocket protocol."

  - "Runner terminal resource API" =/v1/sessions/{id}/resources/terminals[/{id}], backed by
  SessionResourceRegistry/TerminalReegistry.py, runner/app.py:15782).POST launches (or, with ensure_native_terminal: true, routes to _auto_create_claude_terminal,
  app.py:5347); GET looks it up; DELrminal id is deterministic(terminal_resource_id("claude","main")).
  - "Terminal WebSocket protocol" = nals/{id}/attach. Server→client:binary frames of raw PTY bytes. Client→server: JSON text frames ({"type":"resize",...}) or    binary keystroke frames (server/roocstring). In-process it's bridgedby terminals/ws_bridge.py (real forked tmux attach on a PTY) or terminals/control_bridge.py   (tmux control-mode); cross-machineit's multiplexed over one outboundtunnel connection via server/_runner_ws_tunnel.py + runner/transports/ws_tunnel/frames.py     (WSOpenFrame/WSFrame/WSCloseFrame) has outbound connectivity to theserver.                                                                                       
  claude_native_bridge.py is a separate concern from the above: rendezvous between two live     processes — Claude Code sitting inhort-lived "harness turn" processspawned when the web UI submits a message. Its own docstring is explicit: this exists because "Claude's experimental Channels MCinal input path but is blocked atthe org policy layer, so this bridge does not use it" — tmux send-keys/paste-buffer is the    fallback, not the first choice. Thmux.json (socket + target) into abridge dir once the terminal launches (write_tmux_target, called from runner/app.py after     _auto_create_claude_terminal); thele before doing anything.
                                                                                                forwarder / hook / state / status  for codex, cursor, etc.):
  - claude_native_hook.py — the actual Claude Code hook script, injected via --settings <json>  on the CLI invocation (not by mutason). Claude spawns it as asubprocess per hook event (PreToolUse, Stop, UserPromptSubmit, TaskCreated, PreCompact, etc.);it appends events to a bridge-dir ng a synchronous decision(permission requests, ask-user-question, policy evaluation), POSTs to the omnigent server and prints a verdict back to stdout fo
  - claude_native_forwarder.py — background asyncio task that tails those bridge-dir files (hookevents, Claude's own transcript JSta chunks) and POSTs structuredevents (external_conversation_item, external_session_status, etc.) to                         /v1/sessions/{id}/events. Pure rearegister hooks itself.
  - claude_native_state.py — persists the launch cwd per conversation at                        ~/.omnigent/claude-native/<sha256(n, needed because Claude's --resume <sid> requires a matching cwd — a client-side fact, deliberately not sent to the server.      - claude_native_status.py — wraps nd to capture live context-windowusage into the bridge dir, then execs the user's real statusline command so terminal renderingis unaffected.
                                                                                                Generic infra shared across all te_cost_popup.py (tmux popup forbudget approval), native_policy_hook.py (shared PreToolUse/policy-evaluation translation used by both Claude and Codex hooks), native_server_transport.py (thenon-tmux variant used by codex-native's WS JSON-RPC and opencode-native's HTTP/SSE — not used by claude-native itself), native_tal_attach_url builder).
                                                                                                3. Cross-process sync mechanism, s
                                                                                                1. Runner launches the vendor CLI e and publishes a tmux.jsonrendezvous file once ready.                                                                   2. Structured conversation mirroriux at all. For Claude, the hookscript writes events to bridge-dir files that the forwarder tails. For Codex, it's not        file-tailing — codex_native_app_se JSON-RPC-over-WebSocket clientagainst a second, headless "app-server" process omnigent itself spawns, and                   codex_native_forwarder.py does asyer_events(), a live pushsubscription (confirmed zero watchdog/inotify/FSEvents usage anywhere in the package).        3. Either way, the forwarder POSTsnts on the omnigent server.server/routes/sessions.py:5110-5186 (_persist_external_conversation_item) durably appends to  conversation_store (the DB) and pupub/sub keyed by conversation id(runtime/session_stream.py — explicitly no replay buffer, "events emitted before a subscriber connects are lost").
  4. Any connected client holds GET /v1/sessions/{id}/stream (SSE) open and gets live pushes; a reconnecting/new client instead fes/{id} snapshot and dedupes by item id.                                                                                           5. Separately, if a human wants toterminal (not just structuredchat), that's the /attach WebSocket route from part 2, streaming actual tmux pane bytes — a   parallel, independent path from th.
  6. Process bookkeeping: codex_native_process_registry.py keeps a single global                ~/.omnigent/codex-native/process-rtmux_session_name, a unique cmdline tag) guarded by per-launcher flock files under process-owners/. On next launch, a dead        launcher is detected by its releasux session is verified viacmdline-tag match (PID-reuse safety) and reaped with kill-session + SIGTERM on the process    group — so a crashed CLI doesn't l running Codex forever.
  7. Per-session state directories follow one convention package-wide:                          ~/.omnigent/<harness>-native/<sha2for codex, claude, opencode, and by pattern for others) holding client-local facts (launch cwd, bridge sockets, dead-letter       queues for undelivered events) thats but shouldn't leak to theserver.                                                                                       
  4. Detach/reattach — "device follows you"                                                     
  - remain-on-exit on is what makes the tmux session outlive a detaching client at all.         - The CLI's own attach loop distinD (claude_native.py:1970-1998). OnDETACHED it explicitly does not close the terminal resource and does not tear down the runner — comment: "the daemon owns the rurendering it" — so the process andits tmux pane are left running, owned by the runner/daemon rather than the CLI.               - Same-machine fast path attaches  the tmux socket; cross-machinefalls back to the WS PTY bridge over the tunnel.                                              - A phone or browser reattaching g /attach route and, since tmuxnatively supports multiple simultaneous clients per session, lands on the same live pane —    genuine session mirroring (owner =d-only, enforced by LEVEL_OWNER vsLEVEL_READ in terminal_attach.py), not a state-replay trick.                                  - Resume is two-tiered: _resume_pi(lists sessions, no tmux contact);the downstream native module then tries a warm reattach to a live tmux session first, falling back to a cold resume (fresh termin --resume against its transcript)only if none is live.                                                                        
  5. Sandboxing                                                                                
  tmux itself is always unsandboxed, host-side (terminal.py:1036). Only the inner CLI process   launched inside the pane gets wrapr script becomes the pane's shellcommand (terminal.py:1063-1090 → inner/sandbox.py, dispatching to bwrap_sandbox.py on Linux o seatbelt_sandbox.py on macOS). A s exists purely because tmux'scontrol socket is a live coordination channel: with_denied_unix_sockets                       (terminal.py:1941-1949) blocks therom reaching back into theunsandboxed tmux server (comment: it could otherwise tmux -S <sock> run-shell against its own controller) — implemented as a binth on Linux and an explicit denyrule on macOS, plus stripping OMNIGENT_TMUX_SOCK from the inherited environment as defense in depth.
                                                                                                6. What breaks without tmux
                                                                                                Every omnigent <harness>-native coe without tmux: a localshutil.which("tmux") preflight raises click.ClickException before anything launches (present  in all ten *_native.py modules), ahe runner-sidecreate_terminal_instance hard-fails with RuntimeError("tmux is not installed or not on PATH") — enforced independently on both te runner machine. Native-wrappermode is also rejected outright on Windows for the same PTY reason. Critically, any            agent-granted sys_terminal_launch  which harness is driving theconversation — funnels through this same choke point, so tmux gates general terminal          capability, not just the native wr
                                                                                                What is unaffected by tmux's absenses (claude-sdk, openai-agents,cursor, antigravity, copilot, open-responses), CLI_SUBPROCESS (the plain, non--native         codex/pi/kimi/hermes), and ACP_SUB— these have Resume.COLD_ONLY orapp-server-level resume, i.e. transcript replay rather than live-process reattachment, and    never touch tmux. One instructive s classified NATIVE_SERVER (itsactual model-driving protocol is HTTP/SSE, not send-keys injection), yet its CLI wrapper stil hard-requires tmux — purely for miewable/attachable pane. That's aclean natural experiment showing tmux's role split in two: driving the model doesn't need it, but the live-viewing/attach/contin
                                                                                                Verdict: coordination layer, not " scoped, not universal
                                                                                                Within the native-wrapper subsystehitectural work, not incidentalterminal display:                                                                            
  1. It's the input-delivery mechanism where no API exists. Claude Code's own MCP input channel is blocked at the org policy layercstring), so send-keys/paste-buffer is the only way web-UI messages reach the running CLI. This isn't convenience — it's the sole path.
  2. It's the continuity substrate for "device follows you." remain-on-exit + native            multi-client attach is what lets ae reattach to the same live process with zero bespoke state-transfer protocol — that's a session, not a snapshot, being handed    across devices.
  3. It's a security boundary worth defending. The sandbox model treats the tmux control socket as sensitive enough to explicitly  tmux is a live coordinationchannel between processes at different trust levels, not inert scrollback.                    4. Isolation is deliberate: one prminal (random socket, not a sharedmultiplexer instance) — this is closer to "tmux used as a lightweight per-session PTY-hosting daemon" than to a human multiplexi
                                                                                                But it is explicitly not how strucsyncs across the platform — that'sa completely separate hook/forwarder → HTTP POST → DB + SSE pub/sub pipeline that never       touches tmux, and most of omnigentLI-subprocess/ACP modes) runs withno tmux involvement at all. So the accurate framing for the walkthrough: tmux is a            load-bearing coordination layer sp_TUI wrapper family (sessionhosting, input injection, live multi-device viewing, detach/reattach), sitting alongside — no underneath — omnigent's general-pucoordination plane that the restof the platform runs on.                                                                     
  Uncertainties (from the four parallel research passes)                                       
  - The 8 per-harness _run_tmux/buffer-paste implementations were confirmed structurally        identical in the one fully-read fipy) and via docstrings/grep onothers (hermes, goose, antigravity) — not diffed byte-for-byte across all ten.                - runner/app.py's _auto_create_claed-line cold-resume/ucode-authflow) and the NativePaneReaper wiring in the same file were read only enough to confirm call  sites, not exhaustively.
  - Whether session_stream.py's in-process SSE pub/sub has a cross-replica fan-out layer (Redis Postgres LISTEN/NOTIFY) for multi-s was not found or confirmed —plausible gap in a horizontally-scaled deployment, not verified either way.                   - The "daemon" that is said to "owch is referenced by comments inclaude_native.py but its concrete source module wasn't located in this pass.                  - Whether plain codex (non-native,ed Resume.WARM_REATTACH maps to aliteral live-process reattach or just its app-server's own resume concept (no PTY) wasn't     resolved — a taxonomy vs. mechanisead if it matters for thewalkthrough.                                                                                  - Not verified whether native-wrapd) keeps working if only theterminal-attach feature is broken by missing tmux — plausible given how the two paths are     architecturally separate (section directly traced end-to-end.
                                                                                                All four reports are in — the pictg some genuinely great findings(each terminal gets its own private tmux server; the sandbox explicitly firewalls the tmux    control socket; Claude's input pate MCP channel is policy-blocked).Now I'll build the walkthrough page.                                                         
Design plan — Subject: a terminal-native orchestration platform, so the page leans            mono-forward. Palette: warm amber-mber "phosphor" accent reserved for the authoritative node and tunnel paths; five tier colors (client/server/host/runner/harness) used consistently across every diae: monospace display forheadings/labels (the subject's own vernacular), system grotesk for running prose at ~70ch. Layout: left rail nav, hand-drawn , mermaid forsequence/class/component diagrams, <details> progressive disclosure for evidence tables and deep dives, and explicit "unverifiis visually distinct from fact.