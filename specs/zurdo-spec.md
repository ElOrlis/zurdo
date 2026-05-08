# Zurdo — Feature Spec

> A Rust rewrite of [paullovvik/ralph-loop](https://github.com/paullovvik/ralph-loop).
> Goal: preserve the simplicity of the original CLI while adding targeted improvements.
> Intentionally excludes: GitHub Projects v2, mcpls, sync systems.

-----

## Features

### 1. Dependency Graph

Tasks declare dependencies on other tasks. Ralph topo-sorts the queue at load time and skips any task whose dependencies haven’t passed yet.

**Behavior:**

- Blocked tasks are marked `status: blocked` with a `blocked_by` list in state
- Cycles, self-dependencies, and dangling references are caught at validation
- `--analyze` surfaces the resolved execution order before running

-----

### 2. `.zurdo/` State Directory

All generated state for a PRD lives under `.zurdo/<slug>/` at the repo root instead of alongside the PRD file.

**Layout:**

```
.zurdo/<basename>-<hash4>/
├── prd.json
├── progress.txt
├── progress-<timestamp>.txt   # archived logs from prior runs
└── .source                    # repo-relative PRD path (collision sentinel)
```

**Notes:**

- Slug is deterministic: `<basename>-<hash4>` where `hash4 = sha1(repo-relative path)[0..4]`
- `.zurdo/` should be added to `.gitignore` (Zurdo prints a one-time hint but does not modify it)
- `--migrate-state` moves legacy sibling JSON / cwd progress files into the new layout

-----

### 3. Independent Verification

Ralph (not Claude) verifies acceptance criteria after each iteration using inline type hints.

**Supported check types:**

|Type         |Syntax                                |Pass condition         |
|-------------|--------------------------------------|-----------------------|
|`shell`      |``[shell: <cmd>]``                    |Exit code 0            |
|`http`       |``[http: <METHOD> <url> -> <status>]``|Matching HTTP status   |
|`file-exists`|``[file-exists: <path>]``             |Path exists            |
|`grep`       |``[grep: "<regex>" in <path>]``       |Regex matches in file  |
|`manual`     |no hint or ``[manual]``               |Skipped — advisory only|

**Notes:**

- Criteria run concurrently per-task via `tokio` (parallelism over bash’s sequential approach)
- `--analyze` flags criteria with no type hint and suggests rewrites

-----

### 4. Per-Task Git Branching & PRs

Each task gets its own branch and a draft PR. Iterations commit with structured trailers.

**Branch naming:**

```
zurdo/<prd-slug>/<task-id>-<title-slug>
```

**Commit format:**

```
<task-id>: <title>

Iteration <n>. Criteria: <x>/<y> passing.

Zurdo-Task-Id: <task-id>
Zurdo-Status: in-progress | passed | failed
```

**Behavior:**

- Draft PR opens after the first commit
- PR flips to ready-for-review when all criteria pass
- Failed iterations still commit so history stays complete
- `--no-branch` skips all branching and PR activity

-----

### 5. Task-Level Retry

Each task has its own retry limit instead of a single global iteration cap.

**Behavior:**

- `max_attempts` is set per-task in the PRD (with a configurable default)
- A task that exhausts its attempts is marked `status: failed` and skipped
- Other tasks continue unaffected — a stuck task doesn’t burn the whole run
- Attempt count and last error are recorded in state for `--report`

-----

### 6. Model Selection by Effort Level

The model used for each task is determined by its declared effort level, keeping costs proportional to task complexity.

**Default mapping:**

|Effort  |Model            |
|--------|-----------------|
|`low`   |`claude-haiku-*` |
|`medium`|`claude-sonnet-*`|
|`high`  |`claude-opus-*`  |

**Behavior:**

- Effort is declared per-task: `**Effort**: low | medium | high`
- Default mapping is overridable in `.ralph/config.toml`
- If no effort is declared, falls back to a configurable default (e.g. `medium`)
- Model selection composes with provider selection (Feature 7) — effort sets the tier, provider determines which model fills that tier

-----

### 7. Multi-Provider LLM Support

Zurdo supports multiple LLM providers by shelling out to their respective CLIs — no API calls, no SDKs. Each role can be assigned a different provider, decoupling cost and capability decisions from any single vendor.

**Supported providers (v1):**

|Provider      |CLI binary  |Notes                   |
|--------------|------------|------------------------|
|Anthropic     |`claude`    |Claude Code CLI         |
|GitHub Copilot|`gh copilot`|Via GitHub CLI extension|

All execution is via `std::process::Command` — Zurdo shells out exactly as the original did with `claude`, just generalized per role. Additional providers (Gemini, Codex) are planned for future releases.

**Roles:**

|Role      |Description                                                      |
|----------|-----------------------------------------------------------------|
|`executor`|Drives each task iteration (the main agent)                      |
|`verifier`|Reviews failed criteria and suggests fixes (optional second pass)|
|`analyzer`|Powers `--analyze` PRD quality feedback                          |
|`reporter`|Powers `--report` summarization                                  |

**Configuration (`.zurdo/config.toml`):**

```toml
[roles]
executor = { provider = "claude", model = "claude-sonnet-4" }
verifier = { provider = "copilot", model = "gpt-4o" }
analyzer = { provider = "claude", model = "claude-haiku-4" }
reporter = { provider = "claude", model = "claude-haiku-4" }

[effort_map]
low    = { provider = "copilot", model = "gpt-4o-mini" }
medium = { provider = "claude", model = "claude-sonnet-4" }
high   = { provider = "claude", model = "claude-opus-4" }
```

**Behavior:**

- Zurdo shells out to the provider’s CLI binary for every call — same pattern as the original `claude --print`
- Each role resolves its provider and model independently
- CLI flags override config for one-off runs (e.g. `--executor claude --executor-model claude-opus-4`)
- If a role has no explicit config it inherits the `executor` provider as a fallback
- Zurdo checks that required CLI binaries are on `PATH` at startup and fails fast with a clear error if not

-----

### 8. Skills

Skills are reusable prompt fragments that get injected into the agent context before each task iteration — modeled after how Claude Code loads `.claude/` skills. They allow you to encode project conventions, domain knowledge, or coding standards once and have Zurdo apply them automatically.

**Directory layout:**

```
.zurdo/skills/
├── global/
│   ├── rust-conventions.md     # applied to every task
│   └── test-style.md
└── <prd-slug>/
    └── auth-context.md         # applied only to tasks in this PRD
```

**How skills are loaded:**

- On each iteration Zurdo reads all `.md` files from `skills/global/` and the PRD-specific `skills/<prd-slug>/` directory
- Skill content is prepended to the prompt as a system-level context block
- Skills are hot-reloaded each iteration — edit a skill mid-run and it takes effect immediately
- A `**Skills**: skill-name` field on a task overrides which skills are injected for that task specifically

**Skill file format:**

```markdown
---
name: rust-conventions
applies_to: executor, verifier
---

Always use `thiserror` for error types. Prefer `anyhow` in binaries, `thiserror` in libraries.
Avoid `unwrap()` outside of tests. All public functions must have doc comments.
```

**Behavior:**

- `applies_to` in frontmatter controls which roles receive the skill (defaults to all roles)
- `--list-skills` prints all loaded skills and which tasks they apply to
- `--no-skills` disables skill injection for a run

-----

### 9. Monitoring

Zurdo records structured telemetry for every run locally, with opt-in external export.

**Local monitoring (default):**

All metrics are written to `.zurdo/<slug>/metrics.jsonl` — one JSON record per event, append-only.

```jsonl
{"event":"task_start","task_id":"task-1","provider":"claude","model":"claude-sonnet-4","attempt":1,"ts":"2026-05-08T10:00:00Z"}
{"event":"criteria_check","task_id":"task-1","criterion":"shell: npm test","passed":false,"duration_ms":1200,"ts":"2026-05-08T10:00:03Z"}
{"event":"task_complete","task_id":"task-1","attempts":2,"duration_ms":45000,"criteria_pass_rate":1.0,"ts":"2026-05-08T10:00:45Z"}
{"event":"run_summary","total_tasks":5,"passed":4,"failed":1,"total_duration_ms":180000,"ts":"2026-05-08T10:03:00Z"}
```

**`--report` surfaces:**

- Per-task attempt counts, pass rates, and wall-clock duration
- Criteria hotspots (criteria that failed most across attempts)
- Provider/model usage summary across the run
- Stalled task detection (tasks that hit max attempts without progress)

**External export (opt-in):**

- `--metrics-out stdout` emits the same JSONL to stdout for piping to external tools
- Planned: OpenTelemetry trace export as a future addition

**Configuration (`.zurdo/config.toml`):**

```toml
[monitoring]
enabled = true              # default: true
metrics_out = "file"        # "file" | "stdout" | "both"
retention_runs = 10         # how many archived progress logs to keep
```

-----

## Open Questions

### Effort Definition

**How is effort declared?**

- a) Explicit field in PRD only (`**Effort**: low|medium|high`)
- b) Auto-inferred from criterion count and type
- c) Both — explicit overrides auto-inferred

*Explicit is simpler and more predictable. Auto-inference could be a future addition.*

-----

### Model Mapping

**Where is the model mapping configured?**

- a) Hardcoded defaults only, overridable via CLI flags
- b) `.zurdo/config.toml` at repo root (recommended — survives model releases without code changes)
- c) Per-PRD override field

-----

### Retry Escalation

**Should repeated task failures escalate the model automatically?**

Example: a `medium` task that fails twice bumps from `sonnet` → `opus`.

- a) No escalation — model is fixed to effort level
- b) Opt-in via flag (`--escalate-on-retry`)
- c) Always escalate (one step per N failures, configurable)

*Powerful heuristic but adds complexity. Likely best as an opt-in flag.*

-----

### Retry Limit Default

**What is the default `max_attempts` per task?**

- Suggested default: `5`
- Should it be overridable globally (CLI flag) and per-task (PRD field)?

-----

### Provider Abstraction Layer

**How should the CLI invocation be abstracted?**

- a) Simple trait with `invoke(prompt: &str) -> Result<String>` — each provider implements it as a shell-out, Zurdo doesn’t care about internals
- b) Richer interface that captures stdout/stderr separately, exit codes, and timing per call for better error reporting
- c) Start with (a), add (b) if debugging needs arise

*Option (a) is the right starting point — mirrors the original design exactly, just generalized.*

-----

### CLI Output Parsing

**`claude` and `gh copilot` have different output formats — how is the response extracted?**

- `claude --print` outputs clean text to stdout
- `gh copilot suggest` is interactive by default; needs flags or a wrapper to get plain output

Options:

- a) Per-provider output parser (strip known formatting artifacts per binary)
- b) Require `--no-formatting` / plain output flags where available, fall back to raw stdout
- c) Configurable output regex per provider in `.zurdo/config.toml`

-----

### Effort Map + Provider Interaction

**When both effort level and a role-specific provider are configured, which wins?**

Example: `executor` is set to `claude/sonnet`, but `effort_map.high` is set to `copilot/gpt-4o`. For a `high` task, which applies?

- a) Role config always wins — effort map is only a fallback
- b) Effort map always wins for the executor role
- c) Effort map wins, but role config sets the provider; effort sets the model tier within that provider

-----

### Dependency Branch Merging

**When a task’s dependencies complete, should their branches be merged in before the agent’s turn?**

- Matches ElOrlis behavior but adds git complexity
- On conflict: abort merge, mark task blocked, move on?
- Or keep it simple and leave merging to the developer?

-----

### Skills Scope

**Should skills be scoped further than global / per-PRD?**

- a) Global and per-PRD only (as specced)
- b) Also per-task — a task can declare `**Skills**: skill-name` to load a specific skill
- c) Both — per-task overrides, but global and PRD-level still apply as a base

-----

### Skills Conflict Resolution

**If a global skill and a PRD-level skill contradict each other, which wins?**

- a) PRD-level always overrides global
- b) Both are injected — the agent resolves conflicts
- c) Explicit `priority` frontmatter field per skill

-----

### Monitoring Retention

**How many runs of metrics/progress logs should be kept locally by default?**

- Suggested default: last 10 runs
- Should `--report` be able to aggregate across multiple archived runs (trend view)?

-----

## Intentionally Excluded

|Feature                  |Reason                                          |
|-------------------------|------------------------------------------------|
|GitHub Projects v2       |Adds `gh` CLI dependency, significant complexity|
|`mcpls` / LSP integration|Experimental, obscure dependency                |
|Copilot agent backend    |Out of scope for a focused CLI                  |
|GitHub sync system       |Defeats the simplicity goal                     |