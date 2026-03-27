# Pipeline CLI Reference

The KantBan CLI (`kantban-cli`) runs a pipeline orchestrator that spawns Claude Code agents (`claude -p`) to process tickets on pipeline columns. Firing constraints are enforced before any agent is spawned.

---

## Installation

```bash
npm install -g kantban-cli
```

Or run without installing:

```bash
npx kantban-cli pipeline <board-id> [flags]
```

Published on npm as **`kantban-cli`**. Requires Node.js 22+.

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `KANTBAN_API_URL` | Yes | API base URL (e.g. `https://api.kantban.app` or `http://localhost:3000`) |
| `KANTBAN_API_TOKEN` | Yes | API token starting with `cb_` — generate from account settings |
| `KANTBAN_PROJECT_ID` | No | Explicit project ID. If omitted, auto-resolved from the board |

---

## Pipeline Command

```bash
kantban pipeline <board-id> [flags]
```

### Flags

| Flag | Default | Description |
|---|---|---|
| `--once` | off | Run one scan cycle, wait for all loops to complete, then exit |
| `--dry-run` | off | Print configuration summary without starting agents |
| `--column <id>` | all | Filter to a single column by ID |
| `--max-iterations <n>` | per-column | Override `agentConfig.max_iterations` for all columns |
| `--max-budget <usd>` | per-column | Override `agentConfig.max_budget_usd` for all columns |
| `--model <model>` | per-column | Override `agentConfig.model_preference` for all columns |
| `--concurrency <n>` | per-column | Override `agentConfig.concurrency` for all columns |
| `--log-retention <days>` | 7 | Days to keep log files |
| `--yes`, `-y` | off | Skip safety confirmation prompt |

CLI flags take precedence over per-column `agentConfig` values.

### Stop Command

```bash
kantban pipeline stop <board-id>
```

Sends SIGTERM to a running persistent pipeline and its child `claude -p` processes.

---

## Column Configuration

A column becomes a pipeline column when it has a linked prompt document and its type is not `done`. The orchestrator uses three column-level fields to control agent behavior.

### promptDocumentId

The document whose content becomes the system prompt for `claude -p`. Create a document first, then link it:

```
kantban_create_document(projectId, spaceId, title: "Design Review Prompt", content: "...")
  → returns document ID

kantban_update_column(projectId, columnId, promptDocumentId: "<document-id>")
```

A column without `promptDocumentId` is not a pipeline column and will never spawn agents.

### goal

Free-text guidance injected into the agent's prompt alongside the document content. Use for column-specific instructions that don't warrant a full document.

```
kantban_update_column(projectId, columnId, goal: "Focus on code review and test coverage")
```

`goal` is optional. Set to `null` to clear.

### agentConfig

A JSON object on the column that controls execution behavior. All fields are optional — unset fields use orchestrator defaults.

```
kantban_update_column(projectId, columnId, agentConfig: { ... })
```

| Field | Type | Default | Description |
|---|---|---|---|
| `execution_mode` | `"kant_loop"` \| `"cron_poll"` \| `"manual"` | `"kant_loop"` | How the column processes tickets |
| `concurrency` | positive integer | `1` | Max simultaneous agent loops per column |
| `max_iterations` | positive integer | `10` | Max Claude invocations per ticket before stopping |
| `max_budget_usd` | positive number \| null | `null` (unlimited) | Per-ticket USD budget cap. Maps to `--max-turns` on `claude -p` |
| `gutter_threshold` | positive integer | `3` | Consecutive iterations with no progress before declaring stalled |
| `model_preference` | string | none (Claude default) | Model ID passed to `claude -p --model` (e.g. `"claude-sonnet-4-6"`) |
| `poll_interval_seconds` | positive integer | — | Interval for `cron_poll` execution mode |
| `worktree.enabled` | boolean | `false` | Spawn each agent in a git worktree for isolation |
| `worktree.path_pattern` | string | — | Worktree directory pattern. `{ticket_number}` is replaced at runtime |

Set `agentConfig` to `null` to clear all overrides and use orchestrator defaults.

---

## Board Configuration

### Circuit Breaker

A board-level safety mechanism. When configured, the `board.circuit_breaker_count` firing constraint subject reads from the target column's ticket count.

```
kantban_update_board(projectId, boardId,
  circuitBreakerThreshold: 5,
  circuitBreakerTargetId: "<escalation-column-id>"
)
```

| Field | Type | Description |
|---|---|---|
| `circuitBreakerThreshold` | positive integer \| null | Threshold count (informational — constraints enforce the actual gate) |
| `circuitBreakerTargetId` | column UUID \| null | Column whose ticket count is exposed as `board.circuit_breaker_count` |

To use this with firing constraints, create a constraint like:

```
subject_type: board.circuit_breaker_count
operator: lt
value: 5
```

This blocks all pipeline columns when the circuit breaker target column reaches 5 tickets.

---

## Firing Constraints Integration

The orchestrator evaluates firing constraints at every spawn point:

1. **Scan cycle** — Before processing a column's tickets. If blocked, the entire column is skipped.
2. **Event-driven spawn** — When a ticket moves into a pipeline column via WebSocket.
3. **Queue drain** — When a loop completes and queued tickets are ready to spawn.
4. **Per-ticket spawn** — Belt-and-suspenders check in every spawn code path.

### Orchestrator-only subject resolution

These subjects resolve differently in the orchestrator vs the API evaluate endpoint:

| Subject | API evaluate | Orchestrator |
|---|---|---|
| `column.active_loops` | Always 0 | Live count from in-memory tracking |
| `column.last_fired_at` | Always 0 (blocks most constraints) | Seconds since last spawn, 999999 if never fired this session |
| `board.total_active_loops` | Always 0 | Sum of all active loops across all columns |
| `ticket.field_value` | Error (fail-open) | Resolved from ticket context |

Use `kantban_evaluate_firing_constraints` for a dry-run preview. The orchestrator uses live runtime state for the actual decision.

### Constraint blocking output

When a constraint blocks, the orchestrator logs:

```
[blocked] Column "Design" (abc123): constraint "Next col under 5" FAILED — resolved=7 lt 5
[scan] Column abc123 (Design): BLOCKED by firing constraints — skipping 3 ticket(s)
```

When a constraint has `notify: true` and blocks, a signal is created on the column.

### Constraint cache refresh

Constraints are cached in memory and refreshed:
- Every 30 seconds during the scan cycle (full column scope re-fetch)
- Immediately on `firing_constraint:created/updated/deleted` WebSocket events

---

## Execution Modes

### `kant_loop` (default)

The orchestrator runs a multi-iteration loop per ticket:
1. Fetch ticket and column context
2. Compose prompt from prompt document + goal + context
3. Invoke `claude -p` with MCP tools available
4. Check for progress via fingerprint diff (comments, field values, column change)
5. If no progress for `gutter_threshold` consecutive iterations → stall and stop
6. If ticket moves to a different column → success, stop
7. Repeat up to `max_iterations`

### `cron_poll`

Single-pass processing on a timer. Each ticket gets one Claude invocation per interval. Configured via `poll_interval_seconds`.

### `manual`

Column is recognized as a pipeline column but the orchestrator does not auto-spawn. Agents are started via the `kantban work` CLI command for interactive single-ticket processing.

---

## Concurrency and Queuing

Each pipeline column has a concurrency limit (default: 1). When a column has more tickets than its concurrency allows:

- Tickets up to the concurrency limit spawn immediately
- Excess tickets enter a per-column FIFO queue
- When a running loop completes, the next queued ticket spawns (if constraints pass)
- Queue state is in-memory — lost on orchestrator restart

---

## Blocker Handling

Before spawning, the orchestrator checks if a ticket has unresolved dependency blockers (ticket links with `blocks` relationship where the blocking ticket hasn't reached a `done` column).

- If blocked → ticket is **deferred** (not queued, not spawned)
- Deferred tickets are re-evaluated every scan cycle and on `ticket:moved` events
- If the blocker check API call fails → ticket is deferred (fail-safe)

---

## Gutter Detection

The orchestrator detects when a Claude agent makes no meaningful progress:

- After each `claude -p` invocation, a fingerprint is captured (comment count, field value count, column)
- If the fingerprint matches the previous iteration → gutter count increments
- When gutter count reaches `gutter_threshold` → loop stops with reason `stalled`
- A signal is created on the ticket so future agents know about the stall

---

## Progress Detection (Fingerprinting)

The orchestrator tracks ticket state via fingerprints:

| Fingerprint field | Counts as progress |
|---|---|
| `column_id` change | Yes — ticket moved (loop exits with `moved`) |
| `comment_count` change | Yes — agent wrote a comment |
| `field_value_count` change | Yes — agent set a field value |
| `signal_count` change | No — excluded to avoid false progress from dependency signals |

---

## MCP Configuration

The orchestrator generates a stable MCP config file per board:

```
~/.kantban/pipelines/<board-id>/mcp-config.json
```

This config is passed to every `claude -p` invocation via `--mcp-config`. It connects the spawned agent to the KantBan MCP server with the same API token.

---

## Logs

Pipeline logs are stored at:

```
~/.kantban/pipelines/<board-id>/logs/
```

Each run creates a timestamped log file. Use `--log-retention <days>` to control cleanup (default: 7 days).

---

## Orchestrator Lifecycle

```
initialize()
  → Fetch board scope (columns, circuit breaker config, ticket counts)
  → Identify pipeline columns (has_prompt=true AND type != done)
  → Fetch column scope for each (agent config, constraints, tickets)

scanAndSpawn() [every 30 seconds]
  → Refresh board scope
  → Re-evaluate deferred tickets (blockers may have resolved)
  → For each pipeline column:
      → Refresh column scope
      → Evaluate column-scope firing constraints
      → If BLOCKED: skip column, log which constraints failed
      → If PASSED: for each ticket → spawnOrQueue()

spawnOrQueue(ticketId, columnId)
  → Re-check firing constraints
  → If under concurrency limit and no unresolved blockers: spawn
  → If at concurrency limit: add to FIFO queue
  → If blocked by dependencies: defer

onLoopComplete(ticketId, result)
  → Remove from active tracking
  → Drain queue (spawn next queued ticket if constraints pass)
  → Write completion comment on ticket
  → Create signal on ticket for stalled/error exits
```

---

## WebSocket Events

The orchestrator connects to the board's WebSocket channel for real-time events:

| Event | Orchestrator behavior |
|---|---|
| `ticket:created` | Evaluate constraints, spawn if column passes |
| `ticket:moved` | Evaluate constraints, spawn if target is pipeline column; re-evaluate all deferred tickets |
| `ticket:updated` | Re-evaluate if ticket was deferred |
| `ticket:deleted` / `ticket:archived` | Clean up tracking state, remove from queues |
| `firing_constraint:created/updated/deleted` | Refresh all column constraint caches immediately |

Falls back to 30-second polling if WebSocket disconnects.
