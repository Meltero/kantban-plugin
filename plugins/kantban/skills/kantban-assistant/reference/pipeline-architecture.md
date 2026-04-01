PIPELINE ARCHITECTURE — VERIFIED REFERENCE
==========================================
Verified against source code 2026-03-31.
All function names, field names, parameters, and flow orders match the actual implementation.


1. SYSTEM OVERVIEW
==================

The pipeline CLI (`kantban pipeline <board-id>`) orchestrates autonomous Claude Code agents
that process kanban tickets through board columns. Three nested control loops govern execution:

  Inner Loop (RalphLoop.run)
    Agent works in a Claude -p subprocess → calls run_gates MCP tool → sees failures →
    fixes code → calls move_ticket → gate proxy intercepts → runs gates → pass/fail →
    agent retries or ticket moves.

  Middle Loop (orchestrator per-ticket)
    Iteration ends → orchestrator runs post-iteration gate snapshot → gutter detection
    on gate deltas → stalled? → advisor invoked with gate history → RETRY_WITH_FEEDBACK /
    RETRY_DIFFERENT_MODEL / RELAX_WITH_DEBT / SPLIT_TICKET / ESCALATE.

  Outer Loop (replanner pipeline-wide)
    Orchestrator tracks escalations, token usage, systemic gate failures, duration →
    threshold crossed → replanner fires → CONTINUE / PAUSE_PIPELINE / ARCHIVE_TICKETS /
    CREATE_SIGNAL / ADJUST_BUDGET / ESCALATE_ALL.

Entry point: packages/cli/src/commands/pipeline.ts:runPipeline()
Orchestrator: packages/cli/src/lib/orchestrator.ts:PipelineOrchestrator


2. THREE-LOOP ARCHITECTURE
==========================

Inner Loop — RalphLoop (ralph-loop.ts)
  One ticket, one column. Iterates up to maxIterations.
  Each iteration: compose prompt → invoke Claude -p → check fingerprint for move →
  run post-iteration gates → compute gutter delta → stuck detection → checkpoint.
  Exit reasons: moved | max_iterations | stalled | stopped | error | deleted

Middle Loop — Advisor (advisor.ts + orchestrator.ts:invokeAdvisorRecovery)
  Fires when inner loop exits with stalled | error | max_iterations.
  Budget: advisorMaxInvocations per ticket per column transit (default 2).
  Actions: RETRY_WITH_FEEDBACK, RETRY_DIFFERENT_MODEL, RELAX_WITH_DEBT, SPLIT_TICKET, ESCALATE.

Outer Loop — Replanner (replanner.ts + orchestrator.ts:onLoopComplete)
  Fires when pipeline-wide triggers are crossed.
  Triggers: escalation_count >= 3, cost_threshold_pct >= 75%, repeated_gate_failure_count >= 3,
            duration_threshold_minutes >= 480.
  Max invocations: 3 non-CONTINUE actions before auto-pause.


3. GATE PROXY INTERNAL FLOW
============================

File: packages/cli/src/lib/gate-proxy.ts (GateProxy class)
File: packages/cli/src/lib/gate-proxy-server.ts (MCP stdio server)

The gate proxy is an MCP server spawned per column as a child process. It exposes three tools
over JSON-RPC on stdin/stdout:

  kantban_run_gates — run column gates, report results (agent-initiated check)
  kantban_move_ticket — run gates THEN forward move to KantBan API (interception)
  kantban_complete_task — run gates THEN forward complete to KantBan API (interception)

Env vars required: GATE_CONFIG_PATH, COLUMN_ID, COLUMN_NAME, PROJECT_ID,
                    KANTBAN_API_TOKEN, KANTBAN_API_URL

handleRunGates(columnName, ticketId?)
  resolve gates for column → filter waived gates (field: gate_waiver) → run gates →
  return { passed: all required passed, results }

handleMoveTicket(move: MoveArgs)
  resolve gates → filter waivers → run gates →
  if required failures > 0 → return GATE_FAILURE with formatted errors + hint
  else → forward move to API via PATCH /projects/:pid/tickets/:tid/move

handleCompleteTask(complete: CompleteArgs)
  same pattern as handleMoveTicket but forwards to POST /projects/:pid/tickets/:tid/complete

Message serialization: messageQueue in gate-proxy-server.ts (not GateProxy class) chains
promises to prevent concurrent double-moves on the MCP stdio dispatcher.


4. EVALUATOR COLUMN FLOW
=========================

File: packages/cli/src/lib/evaluator.ts
File: packages/cli/src/lib/orchestrator.ts:handleEvaluatorVerdict()

Columns with column_type "evaluator" get special treatment:
  - Always heavy tier (full tool access for adversarial review)
  - Adversarial preamble injected into prompt (composeEvaluatorPrompt)
  - Agent uses submit_verdict tool instead of move_ticket
  - Harness parses verdict and controls movement

Adversarial preamble: "You are an ADVERSARIAL REVIEWER. Your job is to find problems..."

parseVerdict(raw: string) → Verdict
  Extracts JSON from LLM output → validates against VerdictSchema
  On parse failure: returns reject with parseFailed=true

resolveVerdictAction(verdict: Verdict) → VerdictAction
  approve → 'forward'
  reject with blockers (or empty findings) → 'reject'
  reject with only warnings/nits → 'forward_with_signals'

handleEvaluatorVerdict() actions:
  forward → move to next pipeline column (by board position) with verdict summary
  reject → move BACK to previous pipeline column with findings as comment
  forward_with_signals → move forward, create signals for each finding

Parse failure circuit breaker: if verdict can't be parsed, ticket is held for manual review
(comment written, no movement).


5. TICKET LIFECYCLE STATE MACHINE
==================================

States a ticket traverses during pipeline processing:

  Backlog (column_id=null)
    → [scanAndSpawn or WS event] → Known
  Known (in knownTickets set)
    → [blocker check] → Deferred (has unresolved blockers)
    → [concurrency check] → Queued (column at capacity)
    → [claimTicket] → Active (loop running)
  Deferred (in deferredTickets map: ticketId → columnId)
    → [blocker resolved via ticket:moved event or re-scan] → spawnOrQueue
  Queued (in loopQueues: columnId → ticketId[])
    → [drainQueue when slot opens] → Active
  Active (in activeLoops: ticketId → {columnId, promise})
    → [loop exits] → onLoopComplete
  onLoopComplete
    → moved: comment + signal + terminal cleanup
    → stalled/error/max_iterations: advisor recovery → retry or terminal
    → stopped: comment + cleanup
    → deleted: cleanup only (no comment)

Terminal cleanup (all exit paths):
  clear advisorBudget → clear checkpoint → worktree merge/cleanup → clear gate snapshots


6. PROMPT COMPOSITION STACK
============================

File: packages/cli/src/lib/prompt-composer.ts:composePrompt()

12 sections assembled in order, each with a token budget:

  Section                    Budget (tokens)  Notes
  -------                    ---------------  -----
  1. system_preamble         800              identity, iteration N/M, progress warning,
                                              available tools, knowledge sharing rules
  2. worktree context        (own section)     ## Git Worktree — injected between preamble and signals
  3. signals                 (unbounded)      guardrails at top (API caps to 5 per non-project scope)
  4. gate_results            500              previous gate pass/fail with output snippets
  5. rejection               500              elevate latest QA REJECTION comment or meta.rejectionFindings
  6. column_prompt           Infinity         NEVER truncated — user's prompt document
  7. lookahead               1000             downstream column criteria
  8. run_memory              1000             cross-agent knowledge
  9. ticket_details          1500             title, description, fields, parent/children,
                                              transition_history (1000), ticket_links
  10. comments               2000             windowed: pinned in full, recent 3 in full,
                                              older compressed to first 100 chars of first line only
  11. transition_rules       500              workflow rules
      dependency_requirements 500             blocker/field requirements
  12. linked_documents       2000             per-doc truncation within budget
  13. metadata               200              iteration, projectId, tool_prefix, column, goal

Token estimation: estimateTokens(text) = ceil(text.length / 4)
Truncation: truncateToTokens(text, max) = text.slice(0, max*4) + '\n[...truncated]'

Comment windowing (windowComments):
  pinned comments → always full
  last 3 unpinned → full
  older unpinned → first line only, truncated to 100 chars (all subsequent lines discarded)

Ticket details section budgeted as whole (1500 tokens total), with transition_history
sub-budgeted at 1000 tokens.


7. ADVISOR DECISION MATRIX
============================

File: packages/cli/src/lib/advisor.ts

Invoked by orchestrator.invokeAdvisorRecovery() when loop exits with failure.
Uses Haiku model, no tools, no MCP ($0.01 budget).

Input context:
  ticket title/number/description, column name, exit reason, iterations, gutter count,
  last error, recent comments (empty — not propagated from loop), field values,
  failure patterns (from latest gate snapshot), remaining budget, model tier,
  escalation models, circuit breaker target, gate history (last 3 snapshots),
  current gate results, trajectory classification, diff stats

Decision guide embedded in prompt:
  Same test failing, different error each time → RETRY_WITH_FEEDBACK
  Same exact error every iteration → RETRY_DIFFERENT_MODEL
  All gates pass but agent didn't move → RETRY_WITH_FEEDBACK ("call move_ticket")
  Gates regressing across iterations → SPLIT_TICKET
  Zero gates pass after max iterations → ESCALATE
  Most gates pass, one stubborn failure → RELAX_WITH_DEBT

AdvisorResponse schema:
  action: RETRY_WITH_FEEDBACK | RETRY_DIFFERENT_MODEL | RELAX_WITH_DEBT | SPLIT_TICKET | ESCALATE
  reason: string
  feedback?: string (for RETRY_WITH_FEEDBACK)
  debt_items?: Array<{type, description, severity}> (for RELAX_WITH_DEBT)
  split_specs?: Array<{title, description}> (for SPLIT_TICKET)

Orchestrator handling per action:

  RETRY_WITH_FEEDBACK
    write feedback as comment → delete from knownTickets → spawnOrQueue(skipCompletingCheck=true)

  RETRY_DIFFERENT_MODEL
    resolve next model from escalation ladder [initial, ...escalation] →
    if already at top → return false (no retry) →
    write escalation comment → store model override → spawnOrQueue

  RELAX_WITH_DEBT
    store debt_items as field value → store gate_waiver field (waived gate names) →
    create debt signal on ticket → propagate upstream debt signal to blocked tickets →
    move to next pipeline column (by board position) with relaxed_with_debt handoff

  SPLIT_TICKET
    create child tickets from split_specs → archive parent ticket →
    (if createTickets fails, parent is NOT archived)

  ESCALATE
    move ticket to circuit_breaker.target_column_id →
    write escalation comment


8. REPLANNER TRIGGER AND ACTION FLOW
======================================

File: packages/cli/src/lib/replanner.ts

Trigger evaluation: shouldFireReplanner(triggers, state) → boolean
  Checks four conditions (any true → fire):
    state.escalatedTickets >= triggers.escalation_count (default 3)
    token usage pct >= triggers.cost_threshold_pct (default 75%)
    any gate failing on >= triggers.repeated_gate_failure_count tickets (default 3)
    state.durationMinutes >= triggers.duration_threshold_minutes (default 480)

PipelineState built by orchestrator.buildPipelineState():
  escalatedTickets: ticket count in circuit_breaker.target_column
  totalTokensIn: from costTracker
  maxInputTokens: from costTracker (default 5M)
  repeatedGateFailures: gate name → count of tickets failing (from snapshot store)
  durationMinutes: (Date.now() - pipelineStartTime) / 60000

Replanner prompt includes: trigger reason, pipeline state, ticket summaries with
gate pass rates, available actions.

ReplannerResponse schema:
  action: CONTINUE | PAUSE_PIPELINE | ARCHIVE_TICKETS | CREATE_SIGNAL | ADJUST_BUDGET | ESCALATE_ALL
  reason: string
  ticket_ids?: string[] (for ARCHIVE_TICKETS)
  signal_content?: string (for CREATE_SIGNAL)
  new_max_input_tokens?: number (for ADJUST_BUDGET — not yet implemented)

executeReplannerAction():
  CONTINUE → no-op
  PAUSE_PIPELINE → set pipelinePaused = true
  ARCHIVE_TICKETS → archive each ticket in ticket_ids
  CREATE_SIGNAL → create signal on every pipeline column
  ESCALATE_ALL → set pipelinePaused = true
  ADJUST_BUDGET → logged as not implemented, treated as CONTINUE

Parse failure fallback: returns PAUSE_PIPELINE with safety reason.
CONTINUE responses do not count toward the 3-invocation limit.


9. GUTTER DETECTION LOGIC
===========================

Gutter = consecutive iterations with no progress. When gutterCount >= gutterThreshold → stalled.

Two detection modes in RalphLoop.run():

  Gate-based (preferred, when onPostIterationGates is wired):
    After each iteration, orchestrator runs gates → GateSnapshotStore.record() computes delta.
    delta_from_previous values:
      improved → reset gutterCount to 0
      same → increment gutterCount by 1 (unless field values changed — skip increment)
      regressed → increment gutterCount by 2
      first_check → no adjustment

    Fallback: if gate check throws, falls back to fingerprint comparison.

  Fingerprint-based (original, when no gates configured):
    Compare TicketFingerprint before/after iteration.
    fingerprintsMatch checks: column_id, field_value_count, comment_count
    (signal_count excluded — dependency-blocked agents inflate it)
    Match → gutterCount++, no match → gutterCount = 0

Post-iteration fingerprint retry:
  If ticket still in same column after Claude exits, retry fingerprint fetch twice
  with postMoveRetryDelayMs (default 1500ms) between retries to handle API propagation lag.

Stuck detection override (if stuckDetection config enabled):
  After gutter check, classifyTrajectory or invoke LLM classifier.
  progressing → reset gutterCount to 0
  spinning → gutterCount += 2
  blocked → exit immediately as stalled


10. COST TRACKING FLOW
========================

File: packages/cli/src/lib/cost-tracker.ts:PipelineCostTracker

Initialized from pipeline.gates.yaml settings.budget (BudgetConfig):
  max_input_tokens: number (positive int)
  max_output_tokens: number (positive int)
  warn_pct: number (1-100, default 75)

record(inv: InvocationRecord):
  Tracks per invocation: ticketId, columnId, model, tokensIn, tokensOut, type
  Aggregates: totalIn, totalOut, per-ticket, per-column, per-model breakdowns
  InvocationType: 'heavy' | 'light' | 'advisor' | 'stuck_detection' | 'orchestrator' | 'replanner'

  Per-column tracking: light_calls, heavy_calls, advisor_calls
    advisor_calls incremented for types: advisor, stuck_detection, replanner

isWarning(): input or output pct >= warn_pct
isExhausted(): totalIn >= max_input_tokens OR totalOut >= max_output_tokens

Budget enforcement in scanAndSpawn():
  if costTracker.isExhausted() → set pipelinePaused = true, return

generateReport(pricing?): formats token summary, per-column, per-model breakdowns.
  If pricing map provided: estimates dollar cost per model.


11. COMPONENT INTERACTION MATRIX
==================================

pipeline.ts → orchestrator
  Creates PipelineOrchestrator with full OrchestratorDeps.
  Wires: startLoop (RalphLoop), dispatchLightCall, invokeAdvisor, invokeReplanner,
  setFieldValue, moveTicketToColumn, createTickets, archiveTicket, getFieldValues,
  appendRunMemory, cleanupWorktree, mergeWorktree, emitPipelineEvent,
  createColumnSignal (replanner broadcasts), upsertColumnSignal (constraint notifications),
  costTracker, gateSnapshotStore, eventEmitter.

orchestrator → ralph-loop
  startLoop() creates RalphLoop with LoopConfig + RalphLoopDeps, calls loop.run().
  LoopConfig: maxIterations, gutterThreshold, model, maxBudgetUsd, worktreeName,
  postMoveRetryDelayMs, lookaheadColumnId, runId, startIteration, startGutterCount,
  onCheckpoint, startFingerprint, stuckDetection, invokeStuckDetection,
  onPostIterationGates, toolRestrictions.

orchestrator → advisor
  invokeAdvisorRecovery() on failure exits. Builds AdvisorInput from colScope, gate store.

orchestrator → replanner
  Checked in onLoopComplete after advisor. Builds PipelineState + ticket summaries.

orchestrator → evaluator
  handleEvaluatorVerdict() for evaluator columns. Parses verdict, resolves action.

orchestrator → constraint-evaluator
  evaluateColumnConstraints() before processing each column in scanAndSpawn.
  Also checked in spawnOrQueue (belt-and-suspenders) and drainQueue.

orchestrator → gate-snapshot
  gateSnapshotStore.record() called from onPostIterationGates callback.
  gateSnapshotStore.getLatest/getRecent for advisor context enrichment.

pipeline.ts → ws-client
  PipelineWsClient connects to API websocket, subscribes to board events.
  Events fed into EventQueue → orchestrator.handleEvent().

pipeline.ts → event-queue
  EventQueue coalesces events by ticketId (latest wins, destructive events preserved).
  Drains at 100ms intervals. Priority events bypass queue and execute immediately.

pipeline.ts → mcp-config
  generateMcpConfig() for base config (kantban MCP server only).
  generateGateProxyMcpConfig() for columns with gates (kantban + gate proxy servers).

pipeline.ts → run-memory
  RunMemory initialized if any column has run_memory=true in agent_config.
  Content injected into prompt via fetchRunMemoryContent callback.

pipeline.ts → cost-tracker
  Created from pipeline.gates.yaml settings.budget.
  record() called in onSessionEnd callback and light call/advisor/replanner dispatchers.


12. GATE CONFIG RESOLUTION
============================

File: packages/cli/src/lib/gate-config.ts

Config file: pipeline.gates.yaml (parsed by js-yaml with JSON_SCHEMA)

GateConfig schema:
  default: GateDefinition[]          — gates that apply to all columns
  columns?: Record<string, ColumnGateOverride>  — per-column overrides
  settings?: GateSettings            — cwd, env, total_timeout, budget, pricing

GateDefinition: { name, run, required (default true), timeout? (e.g. "60s", "5m") }
ColumnGateOverride: { extend (default true), gates: GateDefinition[] }

resolveGatesForColumn(config, columnName):
  1. Find column override by case-insensitive name match
  2. No override → return [...config.default]
  3. Override with extend=false → return [...override.gates]
     (warn if gates array is empty — all enforcement disabled)
  4. Override with extend=true → merge: default gates (minus overridden names) + override gates

resolveGatesForColumnWithWorktree():
  If worktree enabled, prepend synthetic __worktree_merge gate:
    run: `git merge '<integrationBranch>' --no-edit`
    required: true, timeout: 30s

parseTimeout(timeout?):
  "60s" → 60000, "5m" → 300000, undefined/invalid → 60000 (default)

Security: gate.run is operator-authored shell, executed via `sh -c` with full process privileges.
Prototype pollution guard: assertNoPrototypePollutionKeys checks for __proto__, constructor, prototype.


13. EVALUATOR VS WORKER COMPARISON
====================================

                        Worker Column                 Evaluator Column
                        -------------                 ----------------
column_type             start/in_progress/default     evaluator
Invocation tier         auto/light/heavy              always heavy
Prompt                  composePrompt (12 sections)   composeEvaluatorPrompt (adversarial)
MCP tools               full kantban + gate proxy     full kantban + gate proxy
Movement                agent calls move_ticket       harness calls moveTicketToColumn
Verdict mechanism       fingerprint-based detection   submit_verdict tool → parseVerdict
Gutter detection        gate snapshot deltas           same RalphLoop (prompt aims for 1 iteration, but not enforced)
Advisor recovery        yes (on failure exit)         no (verdict controls flow)
Post-completion action  comment + signal + cleanup    handleEvaluatorVerdict → forward/reject/forward_with_signals
Rejection target        N/A                           previous pipeline column (by board position)
Approval target         agent-driven                  next pipeline column (by board position)


14. DATA MODEL
===============

Key types (packages/types/src/):

  TicketFingerprint (pipeline-context.schema.ts)
    column_id: string | null
    updated_at: string (datetime)
    comment_count: number
    signal_count: number
    field_value_count: number

  LoopCheckpoint (pipeline-context.schema.ts)
    run_id: uuid
    column_id: uuid
    iteration: int >= 0
    gutter_count: int >= 0
    advisor_invocations: int >= 0
    model_tier: string
    last_fingerprint?: TicketFingerprint
    worktree_name?: string
    updated_at: string (datetime)

  GateResult (gate.schema.ts)
    name, passed, required, duration_ms, output (max 1MB), stderr,
    exit_code (0-255), timed_out

  GateSnapshot (gate.schema.ts)
    timestamp, iteration, results: GateResult[],
    all_required_passed, delta_from_previous: GateDelta

  GateDelta: 'improved' | 'same' | 'regressed' | 'first_check'

  GateConfig (gate.schema.ts)
    default: GateDefinition[], columns?: Record<string, ColumnGateOverride>,
    settings?: { cwd?, env?, total_timeout?, budget?: BudgetConfig, pricing? }

  BudgetConfig (gate.schema.ts)
    max_input_tokens, max_output_tokens, warn_pct (default 75)

  Verdict (gate.schema.ts)
    decision: 'approve' | 'reject'
    summary: string
    findings: VerdictFinding[]

  VerdictFinding: { severity: 'blocker'|'warning'|'nit', file?, line?, description }

  InvocationType (pipeline-session.schema.ts)
    'heavy' | 'light' | 'advisor' | 'stuck_detection' | 'orchestrator' | 'replanner'

  ExitReason (pipeline-session.schema.ts)
    'moved' | 'stalled' | 'error' | 'max_iterations' | 'stopped' | 'deleted'

  AgentConfig (pipeline-context.schema.ts)
    execution_mode?: 'kant_loop' | 'cron_poll' | 'manual'
    model_preference?, max_iterations?, max_budget_usd?, concurrency?,
    gutter_threshold?, poll_interval_seconds?,
    worktree?: { enabled, path_pattern?, on_move?, on_done?, integration_branch? }
    invocation_tier?: 'auto' | 'light' | 'heavy'
    run_memory?, lookahead_column_id?,
    advisor?: { enabled, max_invocations?, model? }
    checkpoint?, model_routing?: { initial, escalation[], escalate_after? }
    builtin_tools?, allowed_tools?, disallowed_tools?,
    stuck_detection?: { enabled, first_check?, interval? }

  DebtItem (pipeline-context.schema.ts)
    type: 'dropped_criterion' | 'missing_test' | 'missing_functionality' |
          'unmet_requirement' | 'waived_gate'
    description, severity: 'high' | 'medium' | 'low', source_column?


=== SECTION 0: COMPLETE SYSTEM FLOWS ===


0.1 STARTUP FLOW (pipeline.ts:runPipeline)
-------------------------------------------

runPipeline(client, args)
  parseArgs(args)
    positional[0] → boardId (UUID validated)
    flags: --once, --dry-run, --column, --max-iterations, --max-budget,
           --model, --concurrency, --log-retention, --yes/-y
  if !yes && !dryRun → printSafetyWarning() + waitForConfirmation()
  validate pipeline.gates.yaml exists in cwd
    parseGateConfig(yaml) → GateConfig
  create PipelineCostTracker if settings.budget exists
  create GateSnapshotStore
  resolve projectId (env KANTBAN_PROJECT_ID or GET board project)
  generateMcpConfig(apiUrl, token, boardId) → mcpConfigPath
  initialize PipelineLogger
  create PipelineEventEmitter
  wire OrchestratorDeps (see 11. Component Interaction Matrix)
  create PipelineOrchestrator(boardId, projectId, deps)
  orchestrator.initialize()
  initialize RunMemory if any column has run_memory=true
  if dryRun → print config summary, cleanup, return
  set up shutdown handlers (SIGTERM, SIGINT, SIGHUP)
  cleanupOrphanedProcesses(boardId)
  writePidFile(boardId)
  spawnReaper (watchdog)
  create EventQueue + PipelineWsClient
  wsClient.connect() (non-fatal)
  eventQueue.start()
  orchestrator.scanAndSpawn()
  if --once → waitForAllLoops, cleanup, exit
  else → setInterval(30s) { scanAndSpawn + wsClient.tryReconnect }


0.2 ORCHESTRATOR INITIALIZATION
---------------------------------

PipelineOrchestrator.initialize()
  fetchBoardScope(boardId) → BoardScope
    board info, columns[], circuit_breaker, backlog_ticket_count, tool_prefix
  cache as cachedBoardScope
  filter pipeline columns: (has_prompt=true OR type=evaluator) AND type !== 'done'
  for each pipeline column (parallel):
    fetchColumnScope(columnId) → ColumnScope
      column info, prompt_document, agent_config, tickets[], transition_rules,
      signals, field_definitions, firing_constraints, tool_prefix
    build ColumnConfig from agent_config:
      concurrency (default 1), maxIterations (default 10), gutterThreshold (default 3),
      modelPreference, maxBudgetUsd, worktree settings, invocationTier,
      lookaheadColumnId, runMemory, advisor settings (default maxInvocations=2, model=haiku),
      checkpointEnabled, modelRouting, stuckDetection, tool restrictions
    cache column scope
    initialize empty queue for column


0.3 SCANANDSPAWN PER-SCAN CYCLE
---------------------------------

orchestrator.scanAndSpawn()
  if pipelinePaused → return
  if costTracker.isExhausted() → set pipelinePaused, return
  clear knownTickets
  repopulate knownTickets from activeLoops keys + all loopQueues entries
  re-evaluate deferred tickets:
    for each deferredTicket:
      hasUnresolvedBlockers? → still blocked: add to knownTickets
                             → unblocked: delete from deferred, spawnOrQueue(skipKnownCheck=true)
  refreshBoardScope() (fresh ticket counts for constraints)
  clear blockedColumns set
  for each pipelineColumn:
    refreshColumnScope(columnId)
    evaluateColumnConstraints(columnId) → blocked? skip column
    for each ticket in colScope.tickets:
      spawnOrQueue(ticketId, columnId)


0.4 SPAWNORQUEUE DECISION TREE
---------------------------------

spawnOrQueue(ticketId, columnId, skipKnownCheck=false, skipCompletingCheck=false)
  if pipelinePaused → return
  if activeLoops.has(ticketId) → return (already running)
  if spawning.has(ticketId) → return (in-flight spawn)
  if !skipCompletingCheck && completing.has(ticketId) → return (in advisor recovery)
  if !skipKnownCheck && knownTickets.has(ticketId) → return (already processed this scan)
  get ColumnConfig for columnId → if missing, return
  if isColumnBlocked(columnId) → return (belt-and-suspenders constraint check)
  activeInColumn = activeCountForColumn(columnId) (active loops + reservations)
  knownTickets.add(ticketId)
  if activeInColumn >= concurrency → queue.push(ticketId), return
  spawning.add(ticketId)
  reserveSlot(columnId)
  hasUnresolvedBlockers?
    → blocked: spawning.delete, releaseSlot, defer ticket, drainQueue
    → check failed: spawning.delete, releaseSlot, defer ticket, drainQueue
  claimTicket(ticketId)
  startTrackedLoop(ticketId, columnId, colConfig)
  finally: spawning.delete, releaseSlot


0.5 FIRING CONSTRAINT EVALUATION
----------------------------------

File: packages/cli/src/lib/constraint-evaluator.ts:evaluateConstraints()

evaluateColumnConstraints(columnId) → EvalResult | null
  build constraint list from colScope.firing_constraints
  synthetic safety net: if no enabled column.ticket_count constraint with scope=column and
    subject_ref=self/null exists, inject one: subject_type=column.ticket_count, operator=gt, value=0
  buildBoardState():
    columns from cachedBoardScope (id, name, position, column_type, wip_limit, ticket_count)
    active_loops: Map<columnId, count> from activeLoops
    last_fired_at: Map<columnId, seconds> from lastFiredAt timestamps
    circuit_breaker_count: ticket count in circuit_breaker.target_column
    backlog_ticket_count from cachedBoardScope
  evaluateConstraints(constraints, boardState, columnId):
    filter: enabled && scope === 'column', sort by position
    for each constraint:
      resolve subject_type:
        column.ticket_count → resolveColumn(ref).ticket_count
        column.active_loops → active_loops.get(column.id) ?? 0
        column.wip_remaining → wip_limit === null ? 999999 : wip_limit - ticket_count
        column.last_fired_at → seconds since last fire (999999 if never)
        board.total_active_loops → sum of all active_loops values
        board.circuit_breaker_count → from boardState
        backlog.ticket_count → from boardState
        time.hour → UTC hour
        ticket.field_value → error (column-scope only)
      resolveColumn(ref):
        'self'/null → find by columnId
        'next' → sorted by position, next after current
        'prev' → sorted by position, prev before current
        UUID → find by id
      applyOperator(resolved, operator, value):
        eq/neq: numeric comparison when both parseable, else strict equality
        lt, lte, gt, gte: numeric coercion
      fail-closed by default (resolve error → fail unless fail_open=true)
    notify flag: upsert column signal for blocked constraints with notify=true
      uses upsertColumnSignal (content-prefix keyed) to replace rather than accumulate
      createColumnSignal preserved for replanner broadcast signals (append behavior)


0.6 STARTTRACKEDLOOP — TIER CLASSIFICATION
---------------------------------------------

startTrackedLoop(ticketId, columnId, config: ColumnConfig)
  tier classification:
    if evaluator column → 'heavy' (guard before classifyTier, not inside it)
    else → classifyTier({ hasPromptDocument, invocationTier })
      invocationTier === 'light' → 'light'
      invocationTier === 'heavy' → 'heavy'
      invocationTier === 'auto' or unset → check prompt:
        !hasPromptDocument → 'light'
        else → 'heavy'

  if tier === 'light' && dispatchLightCall available:
    dispatchLightCall(ticketId, columnId) → promise
    response action handling:
      move_ticket → moveTicketToColumn → LoopResult{moved}
      set_field_value → setFieldValue → LoopResult{stalled}
      archive_ticket → archiveTicket → LoopResult{moved}
      create_comment → createComment → LoopResult{stalled}
      no_action → LoopResult{stalled}
    track in activeLoops, set lastFiredAt
    attach onLoopComplete handler

  if checkpoint enabled:
    set placeholder promise in activeLoops (30s timeout safety net)
    readCheckpoint(deps, ticketId, columnId)
      → found: resume at checkpoint.iteration + 1, checkpoint.gutter_count, checkpoint.model_tier
      → not found or failed: startLoopWithConfig fresh
    on timeout: release ticket from activeLoops/knownTickets/completing,
      reject placeholder (silently caught), checkpoint read result discarded

  else: startLoopWithConfig directly


0.7 STARTLOOPWITHCONFIG — MODEL RESOLUTION
---------------------------------------------

startLoopWithConfig(ticketId, columnId, config, startIteration?, startGutterCount?,
                    resumeModelTier?, startFingerprint?)

  model resolution priority:
    resumeModelTier (from checkpoint or RETRY_DIFFERENT_MODEL override)
    > config.modelRouting?.initial
    > config.modelPreference

  build LoopConfig:
    maxIterations, gutterThreshold, effectiveModel, maxBudgetUsd,
    worktreeName (from generateWorktreeName if worktree enabled),
    lookaheadColumnId, startIteration, startGutterCount, startFingerprint,
    stuckDetection config

  resolve tool restrictions:
    resolveToolRestrictions(builtinTools, allowedTools, disallowedTools)
    if no restrictions → full tool access (includeMcpConfig: true)
    else → { tools?, allowedTools?, disallowedTools?, includeMcpConfig: true }
    toolRestrictions only assigned to LoopConfig if any field is defined (conditional)

  deps.startLoop(ticketId, columnId, loopConfig) → promise
  track in activeLoops, set lastFiredAt
  attach onLoopComplete + error handler


0.8 RALPHLOOP.RUN() (HEAVY TIER)
-----------------------------------

File: packages/cli/src/lib/ralph-loop.ts:RalphLoop.run()

  let gutterCount = startGutterCount ?? 0
  let lastFingerprint = startFingerprint ?? null
  let cumulativeTokensIn/Out/ToolCalls/DurationMs = 0

  for i = startIteration ?? 1 to maxIterations:
    if stopped → return 'stopped'

    if !lastFingerprint → fetchFingerprint (baseline, 30s timeout)
      404 → return 'deleted'
      error → return 'error'

    fetch ticketContext + columnContext (parallel, 30s timeout each)
      404 → return 'deleted'
      error → return 'error'

    pre-flight: if ticket.column.id !== this.columnId → return 'moved'

    fetch runMemoryContent (non-blocking, '' on error)
    fetch lookaheadDocument (non-blocking, undefined on error)

    composePrompt(columnCtx, ticketCtx, iterationMeta)
      throws if no prompt document → return 'error'

    if stopped → return 'stopped'

    invoke Claude -p subprocess:
      emit onSessionStart
      invokeClaudeP(prompt, {mcpConfigPath, model, maxBudgetUsd, worktree,
                             onStreamEvent, toolRestrictions...})
      accumulate tokens/toolCalls/duration
      emit onSessionEnd

    if exitCode !== 0 → return 'error' with last 200 chars of output

    post-iteration fingerprint check:
      fetchFingerprint → if still in same column (not moved), retry twice with
      postMoveRetryDelayMs (default 1.5s) to catch delayed API propagation
      404 → return 'deleted'

    if column changed → return 'moved'

    gutter detection:
      if gates configured → onPostIterationGates(ticketId, iteration)
        improved → gutterCount = 0
        same → gutterCount++ (unless field_value_count changed)
        regressed → gutterCount += 2
        first_check → no change
        gate check throws → fallback to fingerprint comparison
      else → fingerprint comparison
        match → gutterCount++, no match → gutterCount = 0

    stuck detection (if enabled and shouldCheckStuckDetection(config, iteration)):
      if gate snapshots exist → classifyTrajectory (deterministic)
      else if invokeStuckDetection → LLM-based classification
      else → default progressing
      progressing → reset gutterCount if > 0
      spinning → gutterCount += 2
      blocked → return 'stalled' immediately

    if gutterCount >= gutterThreshold → return 'stalled'

    write checkpoint (fire-and-forget):
      { run_id, column_id, iteration, gutter_count, advisor_invocations: 0,
        model_tier, last_fingerprint, updated_at, worktree_name }

  return 'max_iterations'


0.9 LIGHT CALL DISPATCH
--------------------------

File: packages/cli/src/lib/light-call.ts

composeLightPrompt(ctx: LightCallContext)
  Minimal prompt: ticket number/title, column name, project, tool prefix,
  truncated description (300 chars), field values, available columns, transition rules.
  Asks for single JSON action: move_ticket | set_field_value | create_comment |
  archive_ticket | no_action.

parseLightResponse(raw): validates against LightResponseSchema (action + params + reason).

Invocation in pipeline.ts:
  model: haiku, maxBudgetUsd: $0.01, maxTurns: 3
  tools: '' (strip all built-in), includeMcpConfig: false (no MCP tools)
  Costs recorded as type: 'light'


0.10 PROMPT COMPOSITION (SECTIONS + BUDGETS)
----------------------------------------------

See section 6 above for full detail. Key additional notes:

composePrompt() throws if columnContext.prompt_document.content is falsy.

System preamble includes:
  - agent identity and iteration count
  - progress warning when gutterCount > 0
  - unresolved blocker warning
  - available tool descriptions with params
  - knowledge sharing instructions (signals, run memory, iteration summaries)
  - critical rules (check_transition before move, blockers, handoff, burn-limit)

Worktree context added when worktree.enabled, includes merge instructions.
VALID_BRANCH_RE validates integration branch name.


0.11 GATE SYSTEM — THREE INTEGRATION POINTS
----------------------------------------------

1. Agent-initiated: kantban_run_gates MCP tool
   Agent calls run_gates to check current gate status before attempting move.
   Gate proxy server handles via GateProxy.handleRunGates().

2. Move interception: kantban_move_ticket / kantban_complete_task
   When agent tries to move, gate proxy runs gates first.
   Required gate failures → GATE_FAILURE returned to agent (move blocked).
   All required pass → forward to KantBan API.

3. Post-iteration snapshot: onPostIterationGates callback
   Orchestrator runs gates after each Claude iteration for gutter detection.
   Results stored in GateSnapshotStore. Used for:
   - gutter delta computation (improved/same/regressed)
   - stuck detection trajectory classification
   - advisor context enrichment


0.12 GATE RUNNER EXECUTION
----------------------------

File: packages/cli/src/lib/gate-runner.ts

runGate(gate, options):
  Execute gate.run via `sh -c` (execFile)
  Timeout: gate.timeout parsed, or options.timeoutMs, or 60s default
  maxBuffer: 1MB output cap
  On error:
    timed_out = error.killed === true
    buffer_exceeded = error.code === 'ERR_CHILD_PROCESS_STDIO_MAXBUFFER'
    Annotations appended: [TIMED OUT] or [OUTPUT TRUNCATED]
  Return GateResult: name, passed, required (default true), duration_ms, output, stderr,
    exit_code, timed_out

runGates(gates[], options):
  Sequential execution. If totalTimeoutMs set:
    Check elapsed time before each gate
    Exceeded → skip remaining with [SKIPPED — total timeout exceeded]
    Remaining time reduces per-gate effective timeout

formatGateErrors(results):
  Format failures: gate name, required/advisory, exit code, timed_out, last 20 lines of output.


0.13 GATE CONFIG RESOLUTION
------------------------------

See section 12 above.


0.14 GATE SNAPSHOT DELTA COMPUTATION
---------------------------------------

File: packages/cli/src/lib/gate-snapshot.ts

computeDelta(previous: GateSnapshot | undefined, currentResults: GateResult[]) → GateDelta
  no previous → 'first_check'
  currPassing > prevPassing → 'improved'
  currPassing < prevPassing → 'regressed'
  same pass count → check failing gate output diffs:
    for each currently failing gate, find same-named gate in previous failing set
    (only gates present in BOTH failing sets are compared — new/disappeared gates ignored)
    if any matched gate's output differs → 'improved' (agent making different progress)
  else → 'same'

GateSnapshotStore:
  record(ticketId, iteration, results) → GateSnapshot
    Computes delta from previous, creates snapshot, appends to per-ticket list.
    Cap: MAX_SNAPSHOTS_PER_TICKET = 100 (oldest pruned via splice)
  getLatest(ticketId) → last snapshot
  getRecent(ticketId, count) → last N snapshots
  clear(ticketId) → delete all snapshots
  getAllTicketIds() → all tracked ticket IDs


0.15 STUCK DETECTION — TRAJECTORY CLASSIFICATION
---------------------------------------------------

File: packages/cli/src/lib/stuck-detector.ts

shouldCheckStuckDetection(config, iteration):
  firstCheck default 3, interval default 2
  if interval <= 0 → only check at firstCheck
  else → check at firstCheck, then every interval iterations after

classifyTrajectory(snapshots: GateSnapshot[]) → StuckPattern
  Take last 3 snapshots. Extract deltas. Filter out 'first_check'.
  No meaningful deltas → progressing (confidence 0.5)
  All 'same' → spinning (confidence 1.0, "identical gate results")
  All 'regressed' or 'same' → regressing (confidence 1.0, "gate results degrading")
  Some 'improved':
    last meaningful === 'improved' → progressing (1.0, "gate results improving")
    has improved AND regressed but last !== improved → spinning (1.0, "oscillating")
    has improved but none recent → spinning (0.8, "stale improvement")
  Default → spinning (1.0, "no improvement detected")

In RalphLoop.run(), stuck detection remaps:
  regressing → treated as 'spinning' (gutterCount += 2)

Legacy LLM-based classification (composeStuckDetectionPrompt + parseStuckDetectionResponse):
  Haiku call with $0.01 budget, no tools, no MCP.
  Classifies as progressing/spinning/blocked with confidence 0-1.


0.16 ONLOOPCOMPLETE — POST-LOOP PROCESSING
----------------------------------------------

orchestrator.onLoopComplete(ticketId, columnId, result: LoopResult)
  Mark completing.add(ticketId) → activeLoops.delete(ticketId)

  Emit session-end event (layer: session)
  Emit gate events from finalGateSnapshot (layer: gate, per-gate)
  Emit cost event (layer: cost)

  if evaluator column → handleEvaluatorVerdict() → drainQueue → terminalCleanup → return

  if failure exit (stalled/error/max_iterations):
    invokeAdvisorRecovery()
    if retried → return (advisor re-spawned loop, skips drainQueue + terminalCleanup;
      completing.delete called inside advisor before return, then again in finally)

  drainQueue(columnId) (H13: minimize slot idle time)

  check replanner triggers:
    if shouldFireReplanner AND invocations < max:
      build ticket summaries + trigger reason
      invoke replanner
      execute action
      CONTINUE doesn't count toward limit
    if >= 3 non-CONTINUE → auto-pause

  write reason-specific comment:
    moved → success message + append to run memory
    max_iterations → manual review needed
    stalled → no progress message
    error → error details
    stopped → externally stopped
    deleted → stopped (no comment written)

  create signal for stalled/error exits

  terminalCleanup:
    delete advisorBudget → clear checkpoint → worktree merge/cleanup → clear gate snapshots

  finally: completing.delete(ticketId)


0.17 EVALUATOR VERDICT HANDLING
---------------------------------

See section 4 above.


0.18 ADVISOR RECOVERY (MIDDLE LOOP)
--------------------------------------

See section 7 above.


0.19 REPLANNER (OUTER LOOP)
------------------------------

See section 8 above.


0.20 EVENT HANDLING (WS + EVENTQUEUE)
----------------------------------------

WebSocket events (pipeline.ts):
  PipelineWsClient connects → subscribes to board after first server message →
  heartbeat ping every 30s.

  WS event mapping:
    firing_constraint:created/updated/deleted → orchestrator.refreshConstraints()
    ticket:created/moved/updated/archived/deleted → map to PipelineEvent, push to EventQueue

EventQueue (event-queue.ts):
  Coalescing by ticketId (Map keyed by ticketId, latest event wins).
  Exception: destructive events (archived/deleted) never overwritten by non-destructive.
  drainRateMs: 100ms — drains one event per cycle.
  pushPriority(): bypass queue, execute immediately, remove from queue to prevent double-processing.

orchestrator.handleEvent(event: PipelineEvent):
  ticket:moved / ticket:created:
    if pipeline column and not blocked → spawnOrQueue(skipKnownCheck=true)
    if blocked → defer ticket
    if ticket:moved → re-evaluate blocked tickets (fetchBlockedTickets) and deferred tickets
  ticket:updated:
    if deferred in pipeline column → re-evaluate (spawnOrQueue)
  ticket:deleted / ticket:archived:
    clean up: activeLoops, knownTickets, deferredTickets, spawning, advisorBudget, queues


0.21 CROSS-COMPONENT DATA FLOWS
----------------------------------

Ticket context: API → pipeline.ts fetchTicketContext → RalphLoop.run → composePrompt
Column context: API → pipeline.ts fetchColumnContext → RalphLoop.run → composePrompt
Board scope: API → orchestrator.refreshBoardScope → buildBoardState → evaluateConstraints
Column scope: API → orchestrator.refreshColumnScope → scanAndSpawn ticket list + config
Fingerprint: API → RalphLoop.run → gutter detection
Gate results: gate-runner.ts → gate-snapshot.ts → ralph-loop.ts (gutter) + advisor.ts (context)
Run memory: RunMemory.getContent → RalphLoop prompt enrichment
  RunMemory.append ← agent via MCP append_run_memory tool
Checkpoint: writeCheckpoint (RalphLoop) → readCheckpoint (startTrackedLoop)
  field name: 'loop_checkpoint'
Cost data: invokeClaudeP → onSessionEnd → costTracker.record → replanner state
Signals: two creation paths:
  upsertColumnSignal → PUT /signals/upsert (constraint notifications — replace by content prefix)
  createColumnSignal → POST /signals (replanner broadcasts — append)
  Composition: API getComposed returns 5 most recent per non-project scope (project: 100)
Events: wsClient → EventQueue → orchestrator.handleEvent → spawnOrQueue/cleanup
Pipeline events: eventEmitter → API → browser (session, gate, cost, advisor, replanner, evaluator)

MCP ticket context (formatTicketScope in packages/mcp/src/tools/pipeline-context.ts):
  Mirrors prompt-composer budgets: comments windowed (last 3 full, older first-line),
  transition_history (1000 tokens), signals (1500 tokens, deduplicated),
  linked_documents (2000 tokens, per-doc truncation), dependency_requirements (500 tokens).
  JSON appendix dropped — tool returns markdown only.


0.22 INVOKECLAUDEP SUBPROCESS LIFECYCLE
------------------------------------------

File: packages/cli/src/commands/pipeline.ts:invokeClaudeP()

  Build args: ['-p', prompt, '--dangerously-skip-permissions',
               '--output-format', 'stream-json', '--verbose']
  Optional: --mcp-config, --model, --max-turns, --worktree,
            --tools, --allowedTools, --disallowedTools

  max-turns calculation: if maxBudgetUsd set → max(1, ceil(maxBudgetUsd * 10))

  spawn('claude', args, { stdio: ['pipe', 'pipe', 'pipe'] })
  track in activeChildProcesses set
  append child PID to manifest file

  StreamJsonParser on stdout:
    count tool_use blocks in 'assistant' events
    capture 'result' events for token usage + final output

  CLAUDE_TIMEOUT_MS = 1 hour
    SIGTERM after timeout → SIGKILL escalation after 5s

  On close: flush parser, resolve with {exitCode, output, toolCallCount, tokensIn, tokensOut}
  On error: resolve with exitCode 1

  stdin immediately closed (prompt passed via -p flag)


0.23 RUN MEMORY
-----------------

File: packages/cli/src/lib/run-memory.ts:RunMemory

  Initialize: create document with template:
    ## Codebase Conventions (optional seedContent)
    ## Discovered Interfaces
    ## Failure Patterns
    ## QA Rejection History

  getContent(): fetch document → if > 500 lines, truncate to last 500 lines
    with '[Run memory truncated — compaction needed]' prefix.
    Returns '' on error or if not initialized.

  append(section, content): serialized write queue (prevents concurrent writes)
    Find section header → insert content before next section or at end.
    Fire-and-forget on error.

  compact(compactedContent): replace full document content.
    Called by orchestrator after summarization.

  needsCompaction(threshold=500): check if line count exceeds threshold.

  Run memory injected into prompt via fetchRunMemoryContent callback in pipeline.ts.
  Additional truncation to 500 lines in the callback (defense in depth).


0.24 CHECKPOINT
-----------------

File: packages/cli/src/lib/checkpoint.ts

  CHECKPOINT_FIELD = 'loop_checkpoint'
  DEFAULT_STALE_THRESHOLD_MINUTES = 600 (10 hours)

  writeCheckpoint(deps, ticketId, checkpoint: LoopCheckpoint):
    await setFieldValue(ticketId, 'loop_checkpoint', checkpoint)
    Function itself awaits internally; callers make it fire-and-forget via .catch().

  readCheckpoint(deps, ticketId, currentColumnId, staleMinutes?):
    getFieldValues(ticketId) → find 'loop_checkpoint' → parse with LoopCheckpointSchema
    Reject if: parse fails, column_id mismatch, age > threshold (default 10h)
    Returns null on any failure. Fully awaited by callers.

  clearCheckpoint(deps, ticketId):
    await setFieldValue(ticketId, 'loop_checkpoint', null)
    Function itself awaits internally; callers make it fire-and-forget via .catch().

  Orchestrator wires checkpoint in startTrackedLoop (read) and onLoopComplete terminal cleanup (clear).
  RalphLoop writes checkpoint at end of each iteration via onCheckpoint callback.


0.25 MCP CONFIG GENERATION
-----------------------------

File: packages/cli/src/lib/mcp-config.ts

  generateMcpConfig(apiUrl, token, boardId):
    Path: ~/.kantban/pipelines/<boardId>/mcp-config.json
    Content: { mcpServers: { kantban: { command, args, env } } }
    Local dev: uses packages/mcp/dist/index.js if exists, else npx kantban-mcp@latest
    Permissions: dir 0o700, file 0o600

  generateGateProxyMcpConfig(apiUrl, token, boardId, gateConfigPath, columnId, columnName, projectId):
    Path: ~/.kantban/pipelines/<boardId>/mcp-config-<columnId>.json
    Content: { mcpServers: { kantban: ..., 'kantban-gates': { command: node, args: [gate-proxy-server.js],
              env: { GATE_CONFIG_PATH, COLUMN_ID, COLUMN_NAME, PROJECT_ID, tokens } } } }

  cleanupMcpConfig(filePath): delete single config file
  cleanupGateProxyConfigs(pipelineDir): delete all mcp-config-*.json in directory

  Config selection in pipeline.ts startLoop:
    if resolveGatesForColumn returns gates → use gate proxy config
    else → use base config


0.26 TOOL SCOPING
-------------------

File: packages/cli/src/lib/tool-profiles.ts

  resolveToolRestrictions(builtinTools?, allowedTools?, disallowedTools?) → ToolRestrictions
    No restrictions set → { includeMcpConfig: true } (full tool access)
    Any restriction set → { tools?, allowedTools?, disallowedTools?, includeMcpConfig: true }

  No presets — user directly configures per-column via agent_config:
    builtin_tools: string (value for --tools flag, '' strips all built-in)
    allowed_tools: string[] (values for --allowedTools flag)
    disallowed_tools: string[] (values for --disallowedTools flag)

  Applied in startLoopWithConfig → LoopConfig.toolRestrictions
  Forwarded to invokeClaudeP via ClaudeInvokeOptions:
    --tools, --allowedTools, --disallowedTools flags

  Special cases:
    Light calls: tools='', includeMcpConfig=false (no tools at all)
    Advisor: tools='', includeMcpConfig=false
    Stuck detection: tools='', includeMcpConfig=false
    Replanner: tools='', includeMcpConfig=false


0.27 COST TRACKING
--------------------

See section 10 above. Additional detail:

  Where costs are recorded in pipeline.ts:
    Heavy iteration: onSessionEnd callback → type 'heavy'
    Light call: after invokeClaudeP returns → type 'light'
    Advisor: after invokeClaudeP returns → type 'advisor'
    Stuck detection: via invokeStuckDetection wrapper (no direct tracking,
      uses tools='' so cost is minimal)
    Replanner: after invokeClaudeP returns → type 'replanner'

  Cost tracker created from pipeline.gates.yaml settings.budget.
  If no budget configured → costTracker is undefined, no budget enforcement.
  Pipeline cost report printed on shutdown and --once completion.


0.28 GRACEFUL SHUTDOWN
------------------------

shutdown(signal) in pipeline.ts:
  clear rescanTimer
  eventQueue.stop()
  wsClient.send pipeline:stopped → 200ms delay → eventEmitter.close() → wsClient.stop()
  eventEmitter.close()
  stop all activeRalphLoops (loop.stop() sets stopped=true)
  killAllChildProcesses (SIGTERM to all tracked children)
  wait up to 5s for children to exit
  if still alive → SIGKILL escalation
  print cost report
  cleanupMcpConfig, cleanupGateProxyConfigs
  killReaper, removePidFile, removeChildManifest
  process.exit(0)

Orphan cleanup at startup (cleanupOrphanedProcesses):
  Kill stale orchestrator from PID file
  Kill children from PID manifest
  Kill stale reaper from reaper.pid
  Kill orphaned 'claude -p' processes matching board's MCP config directory

Watchdog reaper (reaper.ts):
  Spawned as detached child. Polls orchestrator PID. If dead:
  kills manifest children, removes PID files, cleans MCP configs.
