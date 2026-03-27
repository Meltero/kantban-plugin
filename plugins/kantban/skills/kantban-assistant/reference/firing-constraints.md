# Firing Constraints Reference

Firing constraints are pre-fire gates on pipeline columns. They are evaluated before KantBan spawns a Claude agent instance (`claude -p`) to process a ticket. If any enabled constraint fails, the column does not fire — no agent is spawned.

---

## What Firing Constraints Are

A firing constraint is a rule attached to a pipeline column that says "only run an agent here if this condition is true right now." They are evaluated at fire time, not at ticket-move time.

**Key distinctions:**

- **Firing constraints** gate whether a column spawns an agent at all.
- **Transition rules** gate whether a ticket can move between columns.

Use firing constraints when the question is: "Should the pipeline run right now?" Use transition rules when the question is: "Should this ticket move to the next column?"

---

## Subject Types

Each constraint has a `subject_type` that determines what is measured.

| Subject Type | Description | Notes |
|---|---|---|
| `column.ticket_count` | Number of tickets currently in a column | Requires `subject_ref` |
| `column.active_loops` | Number of active pipeline loops for a column | Requires `subject_ref` |
| `column.wip_remaining` | Remaining WIP capacity: `wip_limit - ticket_count` | Requires `subject_ref` |
| `column.last_fired_at` | Seconds since the column last fired | Orchestrator-only |
| `board.total_active_loops` | Total active pipeline loops across all board columns | No `subject_ref` needed |
| `board.circuit_breaker_count` | Number of tickets in the circuit breaker column | No `subject_ref` needed |
| `backlog.ticket_count` | Number of tickets in the project backlog | No `subject_ref` needed |
| `ticket.field_value` | A custom field value on the current ticket | Orchestrator-only |
| `time.hour` | Current UTC hour (0–23) | No `subject_ref` needed |

**Orchestrator-only** subject types (`column.last_fired_at`, `ticket.field_value`) are only available when the pipeline runs via the CLI orchestrator (`kantban pipeline <board-id>`). The API evaluate endpoint returns 0/error for these — see [pipeline-cli.md](pipeline-cli.md) for orchestrator details and installation.

---

## Subject Refs

For column-scoped subject types (`column.ticket_count`, `column.active_loops`, `column.wip_remaining`), a `subject_ref` specifies which column to measure.

| Value | Meaning |
|---|---|
| `self` | The column this constraint belongs to |
| `next` | The next column in the board's column order |
| `prev` | The previous column in the board's column order |
| `<column UUID>` | An explicit column by ID — use for non-adjacent columns |

---

## Operators

| Operator | Meaning |
|---|---|
| `lt` | Less than |
| `lte` | Less than or equal to |
| `gt` | Greater than |
| `gte` | Greater than or equal to |
| `eq` | Equal to |
| `neq` | Not equal to |

---

## Evaluation Logic

**AND only.** All enabled constraints on a column must pass. There are no OR groups — if you need OR logic, model it as separate columns or use a pipeline template with conditional branching.

**Fail-open.** If a resolver encounters a transient error (e.g., the database is temporarily unreachable), the constraint is treated as passed. A constraint failure never blocks work due to infrastructure issues — it only blocks when the condition is definitively false.

**Enabled flag.** Each constraint has an `enabled` field. Disabled constraints are skipped entirely. Use this to temporarily suspend a constraint without deleting it.

---

## Scopes

| Scope | Evaluated | When to Use |
|---|---|---|
| `column` | Once per column per fire event | Standard gate: "should this column fire at all?" |
| `ticket` | Once per ticket (orchestrator-only) | Per-ticket gate: "should this specific ticket be processed?" |

Most constraints use `column` scope. Use `ticket` scope only when you need to gate on per-ticket data (e.g., a custom field value) and the pipeline runs in orchestrator mode.

---

## Notify Flag

When `notify: true` is set on a constraint, KantBan creates a signal when that constraint blocks firing. Use this for constraints that represent meaningful operational states the user should know about.

**Example:** A `board.circuit_breaker_count > 0` constraint with `notify: true` will create a signal when the circuit breaker column has tickets, alerting the user that automated processing has been paused.

Set `notify: false` (the default) for routine flow-control constraints (WIP limits, business hours) where blocking is expected and not actionable.

---

## MCP Tools

| Tool | What It Does |
|---|---|
| `kantban_list_firing_constraints` | List all firing constraints for a column |
| `kantban_create_firing_constraint` | Create a new firing constraint on a column |
| `kantban_update_firing_constraint` | Update an existing constraint (value, operator, enabled, notify) |
| `kantban_delete_firing_constraint` | Remove a constraint permanently |
| `kantban_evaluate_firing_constraints` | Evaluate all constraints for a column right now — returns pass/fail per constraint with resolved values |

**When to use `kantban_evaluate_firing_constraints`:** Before debugging why a column isn't firing, run an evaluation first. It shows exactly which constraints are passing or failing and what values were resolved. This is faster than inspecting constraints individually.

---

## Common Patterns

### Queue limit — don't overload the next column

Block firing if the next column already has 10 or more tickets.

```
subject_type: column.ticket_count
subject_ref:  next
operator:     lt
value:        10
```

Plain English: "Only fire if the next column has fewer than 10 tickets."

---

### Business hours — only fire 9am–5pm UTC

This requires two constraints (both must pass):

```
subject_type: time.hour
operator:     gte
value:        9
```

```
subject_type: time.hour
operator:     lt
value:        17
```

Plain English: "Only fire during business hours (UTC)." Both must pass for the column to fire.

---

### Cooldown — minimum time between fires

Prevent a column from firing more than once per hour:

```
subject_type: column.last_fired_at
operator:     gt
value:        3600      (1 hour in seconds)
```

Plain English: "Only fire if it has been more than 1 hour since the last fire." This is orchestrator-only.

---

### Backlog gate — don't fire when there's nothing to do

Only fire if the project backlog has tickets:

```
subject_type: backlog.ticket_count
operator:     gt
value:        0
```

Plain English: "Only fire if the backlog is non-empty."

---

### WIP headroom — don't fire if self is full

Only fire if the column itself has remaining WIP capacity:

```
subject_type: column.wip_remaining
subject_ref:  self
operator:     gt
value:        0
```

Plain English: "Only fire if this column hasn't hit its WIP limit."

---

### Circuit breaker — halt all automation on errors

Block firing if any tickets are stuck in the circuit breaker column (with notification):

```
subject_type: board.circuit_breaker_count
operator:     eq
value:        0
notify:       true
```

Plain English: "Only fire if no tickets are in the circuit breaker. Alert me if this blocks."

---

## Difference from Transition Rules

| | Firing Constraints | Transition Rules |
|---|---|---|
| **Gates** | Whether a column spawns an agent | Whether a ticket can move between columns |
| **Evaluated when** | Before `claude -p` is invoked | When `kantban_move_ticket` is called |
| **Applies to** | Agent execution | Ticket movement |
| **Scope** | Column-level or ticket-level | Ticket-level |
| **Fail behavior** | Fail-open (transient errors pass) | Fail-closed (errors block the move) |
| **Configured via** | `kantban_create_firing_constraint` | `kantban_set_transition_rules` |

If a user asks "why isn't this ticket moving?" → check transition rules.
If a user asks "why isn't the agent firing?" → check firing constraints.
