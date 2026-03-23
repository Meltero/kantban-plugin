---
description: "Schedule a playbook to run on a recurring cron"
argument-hint: "Playbook name and schedule (e.g. 'close-sprint daily', 'start-sprint every monday')"
---

# /schedule-playbook — Recurring Playbook Execution

Schedule any playbook (built-in or custom) to run automatically on a cron schedule.

## Steps

### 1. Resolve project context

Read `kanban://my/dashboard` to identify the active project. Hold the resolved `projectId`.

### 2. Discover available playbooks

Call `kantban_list_playbooks` with the resolved `projectId`.

If the user provided a playbook name as an argument, match it against the list (case-insensitive, partial match OK). If no match, show the available playbooks and ask which one to schedule.

If the user did NOT provide a playbook name, present the list and ask which one to schedule:

> "Available playbooks:"
> 1. **start-sprint** — Prepare board for a new sprint
> 2. **close-sprint** — Wrap up a completed sprint
> 3. **incident-response** — Create incident ticket structure
> 4. **release-prep** — Generate release notes from completed tickets
> 5. *(any user-defined playbooks)*
>
> "Which playbook would you like to schedule?"

### 3. Collect playbook parameters

Call `kantban_get_playbook` to retrieve the full playbook details including its parameter schema.

For each required parameter that isn't already known (e.g., `boardId` from context), ask the user. For optional parameters, show defaults and ask if they want to override.

Hold the resolved parameters map.

### 4. Determine the schedule

If the user provided a schedule in their argument, parse it into a cron expression. Examples:
- "daily" → `3 9 * * *` (daily at 09:03)
- "weekdays" or "weekdays 9am" → `3 9 * * 1-5`
- "every monday" → `3 9 * * 1`
- "weekly" → `7 9 * * 1` (Mondays at 09:07)
- "every 4 hours" → `3 */4 * * *`
- "friday 4pm" → `7 16 * * 5`

If the user did NOT provide a schedule, ask:
> "How often should this playbook run? Examples: 'daily', 'weekdays at 9am', 'every monday', 'every 4 hours'"

**Offset rules:** Never schedule on :00 or :30. Offset by 3-7 minutes from round times.

If the schedule is ambiguous, confirm with the user before proceeding.

### 5. Create the cron job

Call `CronCreate` with:
- `prompt`: the `run-playbook` MCP prompt
- `schedule`: the resolved cron expression
- `inputs`:
  - `projectId` — resolved in step 1
  - `playbook` — the playbook name resolved in step 2
  - All playbook parameters collected in step 3

### 6. Confirm and warn about session lifetime

After creating the cron, confirm:
> "Playbook **[playbook display name]** scheduled to run [schedule description] while this session is open. Parameters: [list key params]."

**Always include this warning (do not paraphrase — the user must understand exactly what they're getting):**
> **Session-only schedule:** This cron runs inside this Claude Code terminal session. It will fire on schedule as long as this terminal stays open. When you close the terminal (or it disconnects), the schedule is gone — no catch-up runs, no persistence. You'll need to re-run `/schedule-playbook` next session.
>
> For schedules that survive restarts, use **Claude Desktop's scheduled tasks** instead — see the "Desktop Scheduled Task Recipes" section in the plugin README for copy-paste prompts.

## Notes

- This is an in-session timer, not a background service. If the scheduled time passes while the terminal is closed, the run is silently skipped.
- You must re-schedule each time you start a new Claude Code session. There is no saved state between sessions.
- Each cron invocation runs the playbook through the `run-playbook` MCP prompt, which handles dry-run preview and execution.
- **Important:** Scheduled playbook runs execute with `dryRun: false` (live execution) since the user has already confirmed the parameters and schedule. The `run-playbook` prompt will skip the preview step for scheduled invocations.
- Use `/unschedule` to view or cancel running playbook crons in the current session.
- For standup scheduling, use `/schedule-standup`. For health checks, use `/schedule-health`.
- Keep schedules reasonable — very frequent playbook runs may create excessive board changes.
