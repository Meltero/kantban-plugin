# Playbooks Reference

Playbooks are reusable multi-step workflows that Claude can run on demand. They encode common processes — starting a sprint, responding to an incident, closing a sprint — into a repeatable sequence of tool calls.

---

## What Playbooks Are

A playbook is a named sequence of actions that KantBan executes on a board. Each step can create, move, update, or archive tickets. Playbooks are:

- **Reusable** — run the same playbook at the start of every sprint.
- **Previewable** — always run with `dryRun: true` first to see what will happen.
- **Customizable** — built-in playbooks have parameters; user-defined playbooks can be arbitrarily complex.

---

## Built-In Playbooks

### `start-sprint`

Prepares a board for a new sprint.

**Steps:**
1. Optionally creates a new sprint board with standard columns.
2. Sets WIP limits on In Progress and Review columns.
3. Pulls a configurable number of tickets from the backlog into To Do.
4. Archives any stale Done tickets from the previous sprint.

**Parameters:**
- `boardId` — target board
- `sprintName` — sprint label (e.g., "Sprint 14")
- `ticketCount` — how many backlog tickets to pull in (default: 10)

**When to suggest it:** User says "let's start the sprint" or "set up the board for this week."

---

### `incident-response`

Creates the ticket structure for handling a production incident.

**Steps:**
1. Creates a high-priority parent ticket: "[INC] Incident: {description}".
2. Creates sub-tickets: "Investigate root cause", "Implement fix", "Write post-mortem", "Deploy hotfix".
3. Creates a linked document stub for the post-mortem.
4. Moves the parent ticket to In Progress.

**Parameters:**
- `boardId` — board to create tickets on
- `description` — brief incident description
- `severity` — P1/P2/P3 (sets priority on the parent ticket)

**When to suggest it:** User says "we have an incident," "production is down," or "there's a P1."

---

### `close-sprint`

Wraps up a completed sprint.

**Steps:**
1. Archives all Done tickets from the sprint board.
2. Moves any incomplete To Do or In Progress tickets to the backlog.
3. Calls `kantban_get_velocity` to capture throughput metrics.
4. Calls `kantban_retrospective_insights` and returns a summary.

**Parameters:**
- `boardId` — sprint board to close
- `archiveDone` — whether to archive Done tickets (default: true)
- `moveIncomplete` — where to move incomplete tickets: "backlog" or "next-sprint" (default: "backlog")

**When to suggest it:** User says "close the sprint," "sprint's over," or "let's wrap up."

---

### `release-prep`

Prepares a release by gathering completed work and creating release notes.

**Steps:**
1. Searches for all tickets moved to Done since a given date.
2. Generates a changelog from the completed tickets.
3. Creates a release notes document in the project's doc space.

**Parameters:**
- `projectId` — project to release
- `since` — date to gather completed tickets from (e.g., "2026-03-01")
- `tagPrefix` — optional version tag prefix (e.g., "v1.2")

**When to suggest it:** User says "prepare a release," "what did we ship?", or "generate release notes."

---

## Discovering Available Playbooks

**Tool:** `kantban_list_playbooks`

Returns all playbooks available for a given project — both built-in and user-defined. Always call this before suggesting a playbook, so you can confirm it's available and check its parameters.

```json
{
  "projectId": "uuid-here"
}
```

---

## Running a Playbook

**Tool:** `kantban_run_playbook`

**Always use `dryRun: true` first.** Show the user what will happen, then confirm before running.

```json
{
  "playbookId": "start-sprint",
  "boardId": "uuid-here",
  "parameters": {
    "sprintName": "Sprint 14",
    "ticketCount": 12
  },
  "dryRun": true
}
```

The dry run returns a list of actions: "Would create 4 tickets", "Would move 12 tickets from Backlog to To Do", "Would set WIP limit of 3 on In Progress column".

Show this to the user. When they confirm, re-run with `dryRun: false`.

---

## User-Defined Playbooks

Users can create custom playbooks through the KantBan UI or API. Claude should:

1. Call `kantban_list_playbooks` to discover what's available.
2. Never assume a playbook exists — always verify with the list call.
3. If a user mentions a playbook by name that isn't in the list, say so and offer to help them create one or find the right built-in.

User-defined playbooks appear alongside built-in playbooks in the list response. They follow the same `dryRun` preview flow.

---

## Suggesting Playbooks

Offer a playbook when the user's request maps to one:

- "Let's kick off the new sprint" → suggest `start-sprint`
- "Production is down" → suggest `incident-response`
- "Sprint's done, let's clean up" → suggest `close-sprint`

How to offer: "I can run the `start-sprint` playbook — it'll set WIP limits, pull 10 tickets from the backlog, and archive last sprint's Done tickets. Want to preview it first?"

If the user says yes, run with `dryRun: true` and show the plan.

---

## Scheduling Playbooks

Playbooks can be scheduled to run automatically on a cron schedule using the `/schedule-playbook` command.

**How it works:**
1. User invokes `/schedule-playbook` with an optional playbook name and schedule.
2. Claude discovers available playbooks and collects parameters.
3. A `CronCreate` job is created that invokes the `run-playbook` MCP prompt on the specified schedule.
4. Each cron invocation runs the playbook with the pre-configured parameters.

**Example usage:**
- `/schedule-playbook close-sprint every friday` — close the sprint every Friday
- `/schedule-playbook start-sprint every monday 9am` — start a new sprint every Monday morning
- `/schedule-playbook` — interactive: choose playbook and schedule

**Important notes:**
- Scheduled runs execute live (not dry-run) since the user confirmed parameters at scheduling time.
- Use `/unschedule` to view or cancel scheduled playbooks in the current session.

**Session lifetime:** These crons are in-session timers — they only fire while the terminal is open. When the terminal closes, they're gone with no catch-up. Users must re-schedule each session. For persistent automation, direct users to **Claude Desktop's scheduled tasks** and the "Desktop Scheduled Task Recipes" in the plugin README.

**When to suggest scheduling:**
When a user runs a playbook manually and the workflow is inherently recurring (sprint ceremonies, release prep), offer: "Want me to schedule this playbook to run automatically? Use `/schedule-playbook`."
