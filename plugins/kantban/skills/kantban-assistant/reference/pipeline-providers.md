# Pipeline Providers Reference

The pipeline CLI supports multiple agent providers. Each provider wraps a different AI coding CLI — the orchestrator treats them interchangeably via the `AgentProvider` interface. Provider selection is per-column or per-board; different columns in the same pipeline can use different providers.

---

## Available Providers

| Provider | CLI Binary | ID | Stream Format |
|---|---|---|---|
| Claude Code | `claude` | `claude` | stream-json |
| Codex CLI | `codex` | `codex` | jsonl |
| Gemini CLI | `gemini` | `gemini` | jsonl |

---

## Provider Selection

### Resolution Order

The orchestrator resolves providers in this order:

1. **Column-level** — `agentConfig.provider` on the column
2. **Board-level** — `--provider` CLI flag or board `default_provider`
3. **Fallback** — `claude`

Intelligence operations (advisor, replanner, stuck detection) resolve separately:

1. **Board `intelligence_provider`** (CLI flag)
2. **Board `default_provider`**
3. **Fallback** — `claude`

### Setting a Provider

**CLI flag (all columns):**
```bash
kantban pipeline <board-id> --provider codex
```

**Per-column (agentConfig):**
```
kantban_update_column(projectId, columnId, agentConfig: {
  provider: "gemini",
  model_preference: "fast"
})
```

### Model Tiers

Each provider maps three abstract tiers to concrete model IDs:

| Tier | Claude | Codex | Gemini |
|---|---|---|---|
| `fast` | claude-haiku-4-5-20251001 | gpt-5.1-codex-mini | gemini-2.5-flash-lite |
| `default` | claude-sonnet-4-6 | gpt-5.3-codex | gemini-2.5-flash |
| `thorough` | claude-opus-4-6 | gpt-5.4 | gemini-2.5-pro |

Use tier names (`fast`, `default`, `thorough`) in `model_preference` and `model_routing` for portability. Raw model IDs also work but tie the config to a specific provider.

### Preflight

Before the pipeline starts, each provider used by the board is checked:
- Is the binary on PATH?
- Is authentication available?

If preflight fails, the pipeline logs an error and skips columns using that provider.

---

## Capability Matrix

Not all providers support every feature. When a provider lacks native support for a requested feature, it either degrades gracefully (logs a warning, reports in `AgentResult.degradedCapabilities`) or uses an alternative mechanism.

| Capability | Claude | Codex | Gemini | Notes |
|---|---|---|---|---|
| Tool allowlist | CLI flags | degraded | hooks | Codex has no enforcement mechanism |
| Tool denylist | CLI flags | `--sandbox read-only` | hooks | Codex maps write tools to sandbox |
| Builtin tool stripping | `--tools ''` | `--sandbox read-only` | hooks | Codex approximates with sandbox |
| Max turns | `--max-turns` | degraded | hooks + turn file | Codex has no turn limit |
| MCP config injection | `--mcp-config` file | `-c` CLI flags | `.gemini/settings.json` | Each uses a different mechanism |
| Native worktree | `--worktree` flag | manual `git worktree add` | manual `git worktree add` | Codex/Gemini create worktrees before spawn |
| Sandbox modes | none | `--sandbox read-only` | via hooks | Claude doesn't need sandbox modes |

### What "Degraded" Means

When a feature is degraded, the provider:
1. Logs a warning on the first iteration
2. Posts a comment on the ticket listing degraded capabilities
3. Returns the list in `AgentResult.degradedCapabilities`
4. Continues execution without the feature

The pipeline does not fail — it runs with reduced enforcement. This is intentional: a Codex column without turn limits is better than no Codex column at all.

---

## Provider Details

### Claude Code (`claude`)

The reference provider — all pipeline features have first-class CLI support.

**Invocation:**
```
claude -p <prompt> --dangerously-skip-permissions --output-format stream-json \
  --model <model> --max-turns <n> --worktree <dir> \
  --allowedTools <t1> <t2> --disallowedTools <t3> \
  --mcp-config <path>
```

**MCP config:** Writes JSON file to `~/.kantban/tmp/mcp-config-{ts}.json`, passed via `--mcp-config`.

**Tool scoping:** Direct CLI flags — no shims or hooks needed.

**Worktrees:** Native `--worktree` flag; Claude CLI creates the worktree internally.

**No degradation:** All pipeline features are natively supported.

---

### Codex CLI (`codex`)

OpenAI's Codex CLI. Best-effort degradation for unsupported features.

**Invocation:**
```
codex exec --json --dangerously-bypass-approvals-and-sandbox \
  -m <model> -C <worktree-dir> \
  -c mcp_servers.<name>.command=<cmd> \
  -c mcp_servers.<name>.args=<args> \
  <prompt>
```

**MCP config:** Passed via `-c` CLI flags (one per server property). Does NOT use `CODEX_CONFIG_DIR` (causes state file conflicts with concurrent runs).

**Tool scoping:** Limited. Write-capable tools (`Write`, `Edit`, `Bash`, `NotebookEdit`, `shell`, `file_write`, `file_edit`) can be blocked by mapping to `--sandbox read-only`. Per-tool allowlists and denylists are not supported — they degrade.

**Worktrees:** Manual creation via `git worktree add` before spawning. The provider handles creation, reuse, and cleanup transparently.

**Known issue:** MCP tool calls are cancelled on Codex CLI 0.117.0+ (OpenAI issue #16685). Mitigated by setting `CODEX_FEATURE_TOOL_CALL_MCP_ELICITATION=false`.

**Degrades:** `toolAllowlist`, `toolDenylistAdvisory` (emitted only when the denylist contains a writing tool — triggers `--sandbox read-only`), `builtinToolStripping` (maps to sandbox), `maxTurns`.

---

### Gemini CLI (`gemini`)

Google's Gemini CLI. Uses hooks for features not available via CLI flags.

**Invocation:**
```
gemini -p <prompt> --yolo --output-format json --model <model>
```

**MCP config:** Writes `.gemini/settings.json` to a temporary directory, spawns `gemini` with `cwd` set to that directory. Gemini CLI auto-discovers settings from `cwd`.

**Tool scoping:** Implemented via `BeforeToolSelection` hook. A Node.js shim script (`gemini-hooks.mjs`) reads environment variables and filters the tool list:
- `KANTBAN_ALLOWED_TOOLS` — JSON array of allowed tool names
- `KANTBAN_DISALLOWED_TOOLS` — JSON array of disallowed tool names
- `KANTBAN_BUILTIN_TOOLS_MODE` — `""` to strip all builtins (keep only `mcp_*` tools)

**Turn limits:** Implemented via `AfterAgent` hook. The hook maintains a counter in a temp file and signals stop when the limit is reached.

**Worktrees:** Manual creation (same as Codex). Settings file is copied into the worktree directory.

**Exit code quirk:** Exit code 53 is mapped to 0 (expected when hook enforces turn limit).

**Degrades only if:** The hook script (`gemini-hooks.mjs`) cannot be resolved at runtime. If the script is found, all features work.

---

## MCP Config Strategies

Each provider injects KantBan MCP servers differently:

| Provider | Format | Location | Mechanism |
|---|---|---|---|
| Claude | JSON | `~/.kantban/tmp/mcp-config-{ts}.json` | `--mcp-config` flag |
| Codex | CLI flags | inline | `-c mcp_servers.<name>.<key>=<value>` |
| Gemini | JSON | `{session-dir}/.gemini/settings.json` | `cwd` discovery |

All three receive the same two MCP servers:
1. **kantban** — Main KantBan MCP server (with `KANTBAN_HIDDEN_TOOLS` to hide movement tools)
2. **kantban-gates** — Gate proxy MCP server (intercepts movement, enforces gates)

---

## Provider-Specific Configuration Patterns

### Read-Only Reviewer Column

**Claude:**
```json
{ "disallowed_tools": ["Edit", "Write", "Bash"] }
```

**Codex:**
```json
{ "disallowed_tools": ["Edit", "Write", "Bash"] }
```
*Effect: Degrades to `--sandbox read-only` — blocks all write operations, not just the listed tools.*

**Gemini:**
```json
{ "disallowed_tools": ["Edit", "Write", "Bash"] }
```
*Effect: Hook filters these tools from `allowedFunctionNames`.*

### MCP-Only Agent (Strip Builtins)

```json
{ "builtin_tools": "" }
```

- **Claude:** `--tools ''` — strips all builtin tools, keeps MCP tools
- **Codex:** `--sandbox read-only` (approximation — also blocks MCP write tools)
- **Gemini:** Hook keeps only `mcp_*` prefixed tools

### Model Routing Across Providers

Use tier names for provider-portable routing:

```json
{
  "provider": "gemini",
  "model_routing": {
    "initial": "fast",
    "escalation": ["default", "thorough"],
    "escalate_after": 2
  }
}
```

This resolves to `gemini-2.5-flash-lite` -> `gemini-2.5-flash` -> `gemini-2.5-pro`.

The same config with `"provider": "claude"` resolves to `claude-haiku-4-5-20251001` -> `claude-sonnet-4-6` -> `claude-opus-4-6`.

---

## Choosing a Provider

The pipeline doesn't prescribe which provider to use. Consider:

| Factor | Claude | Codex | Gemini |
|---|---|---|---|
| Tool restriction enforcement | Full | Partial (sandbox only) | Full (via hooks) |
| Turn limit enforcement | Full | None | Full (via hooks) |
| Worktree creation | Native (fastest) | Manual | Manual |
| Token pricing | Anthropic rates | OpenAI rates | Google rates |
| Model variety | Haiku/Sonnet/Opus | Codex Mini/Codex/GPT-5.4 | Flash Lite/Flash/Pro |

**Mixed pipelines** are supported: use Claude for columns needing strict tool control, Gemini for cost-sensitive review columns, Codex for OpenAI-ecosystem compatibility.

---

## Adding a New Provider

To add a provider, implement the `AgentProvider` interface:

```typescript
interface AgentProvider {
  readonly id: string;
  readonly displayName: string;
  capabilities(): ProviderCapabilities;
  invoke(request: AgentRequest): Promise<AgentResult>;
  preflight(): Promise<PreflightResult>;
}
```

Required files:
1. `src/providers/<name>-provider.ts` — provider class
2. `src/providers/<name>-parser.ts` — stream output parser (implement `NormalizedStreamEvent` translation)
3. `src/providers/__tests__/<name>-provider.test.ts` — tests
4. `src/providers/__tests__/<name>-parser.test.ts` — parser tests
5. Register in `src/commands/pipeline.ts` (`createProviderRegistry`)
6. Export from `src/providers/index.ts`
7. Add to `mcp-config-strategy.ts` if config format differs
