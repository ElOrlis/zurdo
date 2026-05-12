# Zurdo Spec Enhancement — Interview Decisions

This document consolidates the decisions made during the spec-enhancement interview. It is intended as the input to partitioning the spec into per-feature PRDs. Each section below represents settled design ground; open items at the end flag what still needs work before PRD partitioning.

The decisions are grouped by the three weak areas identified at the start of the interview: **data model**, **runtime semantics**, and **provider/CLI integration**, with a final cross-cutting section on **config**.

---

## 1. Data Model

### 1.1 PRD File Format

PRDs are **pure markdown with a strict, formally specified grammar**. Anything outside the grammar is a parse error with a precise line number and a suggested correction.

**Multiple PRDs per repo.** One PRD is selected per Zurdo invocation via CLI argument. Each PRD has its own state directory under `.zurdo/<slug>/` where `<slug>` is derived from the PRD filename plus a short hash for uniqueness.

**Grammar (informal):**

```markdown
# PRD: <free-form title>

<free-form prose intro — not parsed>

## Task: <task-id> — <task title>

**Effort**: <key from effort_map>
**Depends-on**: [<task-id>, <task-id>, ...] # or `[]` for no deps
**Max-Attempts**: <integer> # optional, falls back to defaults.max_attempts
**Skills**: <skill-name>, <skill-name> # comma-separated, optional
**Agent-timeout**: <duration> # optional, e.g. `30m`, `1h`, `45s` — falls back to timeouts.agent_seconds

### Description

<free-form prose, passed verbatim to the executor>

### Acceptance Criteria

- [ ] <human-readable criterion text> [<hint>] [<hint>] ...
- [ ] ...
```

**Strict rules:**

1. Task headings are H2 with the exact prefix `## Task: <id> — <title>`. The em-dash is the separator. `<id>` must match `^task-[a-z0-9-]+$` and be unique within the PRD.
2. Metadata fields are exactly `**Key**: value` on their own line, in a contiguous block under the H2 (no blank lines between them). Keys are a closed enum: `Effort`, `Depends-on`, `Max-Attempts`, `Skills`, `Agent-timeout`. Unknown keys are errors. `Effort` and `Depends-on` are required; `Max-Attempts`, `Skills`, and `Agent-timeout` are optional and fall back to config defaults (see §4.2).
3. `Depends-on` uses YAML-style array syntax: `[task-1, task-2]`. Empty array `[]` for no deps.
4. Cross-PRD dependencies are not allowed. A `Depends-on` referencing a task not in the same PRD is a validation error.
5. Section headings under each task are H3 from a closed enum: `Description`, `Acceptance Criteria`. Order is enforced.
6. Acceptance criteria are GFM task-list items (`- [ ]`). Trailing `[hint]` blocks are greedily consumed from the end of the line, in source order.
7. Every criterion must have at least one type hint. A criterion with no hint is a validation error. `[manual]` must be explicit if intended.
8. Duration values (`Max-Attempts`, `Agent-timeout`) require explicit units (`s`/`m`/`h`). A bare integer is an error.

**Effort values are a closed enum defined by `effort_map` in config.** `**Effort**: medium` is valid iff `medium` is a key in `effort_map`. The grammar does not hardcode `low | medium | high`.

### 1.2 Acceptance Criteria — Hints and Semantics

**Hint types:**

| Hint                                 | Behavior                                              |
| ------------------------------------ | ----------------------------------------------------- |
| `[shell: <cmd>]`                     | Run shell command; pass iff exit code 0.              |
| `[http: <method> <url> -> <status>]` | Make HTTP request; pass iff response status matches.  |
| `[file-exists: <path>]`              | Pass iff file exists at path (relative to repo root). |
| `[grep: <pattern> in <file>]`        | Pass iff pattern is found in file.                    |
| `[manual]`                           | No machine check. Presence affects task status only.  |

**Multiple hints per criterion** are AND'd — all must pass. Hints run and report in source order. Failure messages identify the specific hint that failed.

**`[manual]` mixed with automated hints:** the manual portion is ignored. Automated hints still gate the criterion. Manual verification is out-of-band, and any defects discovered manually go into a new PRD.

**Manual criteria carry zero machine signal.** Zurdo never reads or writes GFM checkbox state (`- [x]` vs `- [ ]`). Checkboxes remain unticked in the markdown forever; ticking them is at most personal bookkeeping for humans. Zurdo never modifies the PRD file.

### 1.3 Dependencies

**Syntax:** `**Depends-on**: [task-1, task-2]`. YAML-style array. Empty array for no deps.

**Scope:** local to one PRD only.

**Semantics: ordering-only.** A dependency means "this task does not start until its dependencies have reached a terminal-pass state." It does not affect git branch lineage in any logical sense beyond what falls out of the cumulative-branching execution model (see §2.3).

**A dep is satisfied when the dependency reaches `passed` or `passed-pending-review`.** `passed-pending-review` is sufficient because manual review is an out-of-band concern; automated work is done.

### 1.4 Task Status Enum

| Status                  | Meaning                                                                                                       | Terminal? |
| ----------------------- | ------------------------------------------------------------------------------------------------------------- | --------- |
| `pending`               | Not yet attempted.                                                                                            | No        |
| `blocked`               | Waiting on running dependencies.                                                                              | No        |
| `in-progress`           | Currently iterating.                                                                                          | No        |
| `passed`                | All criteria passed; no manual criteria present, or all automated pass and no manual criteria exist.          | Yes       |
| `passed-pending-review` | All automated criteria passed; one or more manual criteria are present (human review obligation outstanding). | Yes       |
| `failed`                | Exhausted `Max-Attempts` without satisfying all automated criteria.                                           | Yes       |
| `blocked-by-dependency` | An upstream dependency reached `failed` or `blocked-by-dependency`. Never runs.                               | Yes       |

`blocked-by-dependency` propagates transitively. Reports distinguish failures from blocked-by-dependency tasks so root causes are clear.

---

## 2. Runtime Semantics

### 2.1 Concurrency

**Sequential execution only in v1.** One task at a time. Parallel execution (`--parallel N`) is a deferred future feature.

**Ordering:** topo-sort the dep graph; among runnable tasks (deps all satisfied, not blocked-by-dependency), pick the one that appeared earliest in the PRD's source order. This makes declaration order a deterministic tiebreaker.

### 2.2 State Persistence

**`prd.json` is the single source of truth.** Git artifacts (branches, commits, trailers) are informational only. Crash recovery, resume, and reporting all read from `prd.json`.

**Location:** `.zurdo/<slug>/prd.json`. Atomic writes via temp-file-and-rename.

**Schema (v1):**

```json
{
  "schema_version": 1,
  "prd_path": "prds/auth.md",
  "prd_hash": "<sha1 of PRD file at last successful parse>",
  "started_at": "2026-05-08T10:00:00Z",
  "last_updated": "2026-05-08T10:15:00Z",
  "tasks": {
    "task-1": {
      "status": "passed",
      "attempts": 2,
      "branch": "zurdo/<slug>/task-1-<slugified-title>",
      "last_commit_sha": "abc123...",
      "passed_at": "2026-05-08T10:10:00Z",
      "pr_url": "https://github.com/org/repo/pull/42",
      "iteration_summary": "Iteration 1 failed on cargo test; iteration 2 added missing import and passed.",
      "iterations": [
        {
          "attempt": 1,
          "started_at": "...",
          "ended_at": "...",
          "agent_exit_code": 0,
          "criteria_results": [
            {
              "hint": "shell: cargo test",
              "passed": false,
              "duration_ms": 1200,
              "exit_code": 101,
              "stdout_truncated": "...(max 4KB)...",
              "stderr_truncated": "...(max 4KB)..."
            }
          ]
        },
        { "attempt": 2, "...": "..." }
      ]
    }
  }
}
```

**Per-criterion result fields are truncated** to 4KB each for `stdout_truncated` / `stderr_truncated`. The `iterations` array is append-only — every attempt's full result set is retained for the report.

**`pr_url`** is populated after successful PR creation (only when `--create-prs` was passed and the task reached a terminal-pass state). Null otherwise. On resume, any terminal-pass task with `pr_url: null` triggers a PR-creation retry. **`iteration_summary`** is generated by the `reporter` role at PR-creation time (and at `zurdo report` time when missing) and cached here so the reporter is not re-invoked on subsequent runs.

### 2.3 Per-Iteration Lifecycle

**Pre-flight check.** Before invoking the agent for a task, Zurdo runs all the task's criteria against the current state of the task's branch (or `main`, if the branch hasn't been created yet). If all pass: mark task `passed` with `attempts: 0`, do **not** create a branch or PR. Skip to the next task.

**Per-iteration steps** (when at least one criterion fails on pre-flight):

1. Ensure task branch exists. If not, create it off the most recently passed task's branch in this run (or `main` if none yet — see §2.4 on cumulative branches).
2. Build the agent prompt (see §3.2).
3. Invoke `AgentCli` against the prompt. Bounded by `Agent-timeout` (per-task) or `timeouts.agent_seconds` (default).
4. Whatever the agent did to the working tree, commit it on the task's branch with a structured trailer (`Zurdo-Status: passed | failed`, etc.). Failed iterations still commit — full audit trail in git.
5. Re-run all criteria (yes, even ones that passed last iteration — regression detection).
6. Record per-criterion results in `prd.json` under this attempt's entry.
7. Increment `attempts` (this happens **only after the commit succeeds**; crashes before this point do not consume budget).
8. If all criteria pass → `passed` (or `passed-pending-review` if manual criteria exist). Else if `attempts >= Max-Attempts` → `failed`. Else → loop.

### 2.4 Cumulative Branches

When a task creates its branch, it branches off the **most recently passed (or passed-pending-review) task's branch in this run**, not `main`. If no task has reached a terminal-pass state yet in this run, branch off `main`.

This produces a deterministic, linear stack of branches. Downstream tasks naturally see upstream tasks' code without any merge logic. Reviewers see a stack of PRs each containing only that task's incremental diff, all targeting `main`.

The `--no-branch` flag (Feature 4 in original spec) opts out of all git side-effects entirely; state still lives in `prd.json`.

### 2.5 Failure and Recovery

**On crash mid-iteration:** next run detects uncommitted changes on the in-progress task's branch → `git reset --hard HEAD` → restart the iteration → `attempts` is **not** incremented (the previous attempt didn't complete a commit).

**On Ctrl-C:**

- First Ctrl-C: finish the current iteration cleanly (commit, update state, exit).
- Second Ctrl-C: hard exit; rely on crash recovery on next invocation.

**On agent timeout:** kill the process group (not just the parent PID), `git reset --hard HEAD`, count as a failed attempt (distinct from a "criteria failed" attempt in metrics).

**On agent exit 0 but criteria fail:** treat as a normal failed iteration. Agent's nonzero exit is informative but not authoritative; criteria are authoritative.

**On PRD hash mismatch at run start:** abort with a structured diff of what changed (tasks added/removed/modified). Two flags to proceed:

- `--reset`: archive old state, start over.
- `--accept-changes`: preserve status of unchanged tasks; new/changed tasks become `pending`.

### 2.6 Criteria Execution

**Working directory:** always at repo root. Users compose `cd subdir && cmd` inside `[shell:]` if a subdirectory is needed. No `**Working-dir**` field.

**Branch state:** Zurdo ensures the task's branch is checked out before running criteria. Criteria reflect branch state, not main.

**Timeouts:** global default in `.zurdo/config.toml` under `timeouts.criterion_seconds` (default 300 = 5 minutes). Applies to `shell` and `http` only. `file-exists` and `grep` are fast enough to not need timeouts. `[manual]` doesn't run. No per-criterion timeout override in v1.

**On timeout:** kill the criterion's process; record as a failed result with a `timed_out: true` flag in the per-criterion result.

---

## 3. Provider / CLI Integration

### 3.1 Two Distinct Traits

Two separate provider traits, deliberately not unified:

```rust
// For the executor role only — agents that modify files.
trait AgentCli {
    fn invoke(&self, prompt: &str, repo_root: &Path) -> Result<AgentRunOutcome>;
}

struct AgentRunOutcome {
    exit_code: i32,
    stdout: String,         // captured for logs/metrics, not parsed for content
    stderr: String,
    duration_ms: u64,
}

// For verifier / analyzer / reporter roles — text completion.
trait CompletionCli {
    fn complete(&self, prompt: &str) -> Result<CompletionOutcome>;
}

struct CompletionOutcome {
    text: String,           // ready-to-use completion content
    raw_stdout: String,     // for debugging
    duration_ms: u64,
}
```

A provider may implement either, both, or neither.

### 3.2 V1 Provider Matrix

| Provider             | Implements `AgentCli` | Implements `CompletionCli` |
| -------------------- | --------------------- | -------------------------- |
| Anthropic (`claude`) | Yes                   | Yes                        |
| GitHub Copilot       | **Cut from v1**       | **Cut from v1**            |

GitHub Copilot is cut from v1 because its CLI surface doesn't actually support the "run an agent against a task" pattern Zurdo needs. The architecture supports additional providers cleanly; adding a real second provider (e.g. Codex, Gemini) is a follow-on.

### 3.3 Executor Prompt Template

Fixed template owned by Zurdo, **not user-configurable in v1**. Sections in fixed order:

```
# Task

<task title>

<task description verbatim>

# Acceptance Criteria

<criteria list, including hint syntax verbatim, with `[manual]` items marked as
"verified by human reviewer">

# Previous Iteration Feedback

<populated on iteration 2+>

The previous iteration produced changes but criteria did not all pass. Failed checks:

- [shell: cargo test --test auth] → exit code 1
  stdout: <truncated>
  stderr: <truncated>
- [file-exists: src/auth.rs] → file not found at expected path
- [http: GET /health -> 200] → got 503; body: <truncated>

Address these failures while keeping previously-passing checks satisfied.

# Instructions

Make the changes needed to satisfy all acceptance criteria. When you believe you are
done, exit cleanly.
```

**Skills are installed into the provider's native skill discovery path, not inlined into the prompt.** See §3.6 for the skill installation model. The prompt template contains no `# Skills` section; the agent finds applicable skills via the provider's own mechanism (e.g., Claude's skill system).

**Failure stdout/stderr in the prompt** is truncated to 4KB each (same limit as `prd.json` storage). Truncation preserves the tail (most recent output) since test failures and HTTP errors typically have the relevant content at the end.

### 3.4 Effort → Model Resolution

The `executor` role's provider is fixed by config. The `**Effort**` field selects a **model within that provider** via `effort_map`. Effort never changes the provider, never changes the role.

```toml
[roles.executor]
provider = "anthropic"
model = "claude-sonnet-4-7"     # fallback only; effort_map normally overrides

[effort_map]
low    = "claude-haiku-4-5"
medium = "claude-sonnet-4-7"
high   = "claude-opus-4-7"
```

Effort_map applies only to the executor. Other roles (`verifier`, `analyzer`, `reporter`) use their role config's `model` directly.

### 3.5 CLI Output Parsing

For `AgentCli`: output is not parsed for content. Zurdo captures stdout/stderr for logs and reads exit code. The "real output" of an agent invocation is the modified working tree.

For `CompletionCli`: per-provider hardcoded parser in Zurdo's source. No configurable regex, no reliance on CLI flags being right.

### 3.6 Skills System

**Storage:** skills live at `.zurdo/skills/<name>/SKILL.md` in **Anthropic-compatible format** (YAML frontmatter with `name`, `description`, optional `allowed-tools`; markdown body; optional sibling files in the same directory).

**Reference:** the `**Skills**: <name>, <name>` task metadata field names skills the executor should have available. A reference to a non-existent skill directory is a **parse-time error** caught by the grammar validator (PRD-01 validator therefore takes the repo root as input, not just a markdown string).

**Installation model:** Zurdo hosts skills in its own location; the `AgentCli` adapter for each provider decides how to make them available to the agent.

```rust
trait AgentCli {
    fn invoke(&self, prompt: &str, repo_root: &Path) -> Result<AgentRunOutcome>;
    fn install_skills(&self, skills: &[Skill], repo_root: &Path) -> Result<SkillInstallOutcome>;
}

enum SkillInstallOutcome {
    Installed,         // skills available to agent via provider's native mechanism
    NotSupported,      // provider has no skill system; skills silently ignored
}
```

**Per-provider behavior in v1:**

| Provider             | `install_skills` behavior                                                                                                                    |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Anthropic (`claude`) | Point Claude at `.zurdo/skills/` via Claude's configuration so it discovers skills natively. No copying, no symlinking. Returns `Installed`. |

**Unsupported providers:** when `install_skills` returns `NotSupported`, Zurdo logs a `warn`-level message per skill ("skill `auth` ignored: provider `xyz` does not support skills") and proceeds. The task may fail for lack of the skill's guidance, but that surfaces as a normal task failure.

**No inlining fallback in v1.** Earlier drafts considered inlining `SKILL.md` contents into the prompt for providers without native skill support. Cut: it complicates the prompt template, doesn't carry sibling files, and creates two divergent skill-authoring conventions. Skill-using PRDs implicitly require a skill-capable provider.

---

## 4. Config

### 4.1 Scope and Resolution

**Per-repo only in v1.** `.zurdo/config.toml` at the repo root. No user-level config, no per-PRD overrides. If shared defaults across repos are needed, the user manages that out-of-band (symlinks, vendoring, etc.).

**Config is required.** A missing `.zurdo/config.toml` is an error. Zurdo provides `zurdo init` to write a sensible default with comments — the same file works as documentation.

### 4.2 Config File Structure

```toml
# .zurdo/config.toml

[roles.executor]
provider = "anthropic"
model = "claude-sonnet-4-7"     # fallback when effort_map doesn't apply

[roles.verifier]
provider = "anthropic"
model = "claude-haiku-4-5"

[roles.analyzer]
provider = "anthropic"
model = "claude-haiku-4-5"

[roles.reporter]
provider = "anthropic"
model = "claude-haiku-4-5"

[effort_map]
low    = "claude-haiku-4-5"
medium = "claude-sonnet-4-7"
high   = "claude-opus-4-7"

[defaults]
max_attempts = 5                # applies when a task omits **Max-Attempts**

[timeouts]
criterion_seconds = 300         # per-check, shell + http only
agent_seconds     = 1800        # applies when a task omits **Agent-timeout**

[providers.anthropic]
cli        = "claude"           # binary on PATH
extra_args = []                 # passed verbatim on every invocation
```

### 4.3 Pre-flight Validation

At run start, Zurdo validates:

1. `.zurdo/config.toml` exists and parses.
2. Every role's `provider` exists in the `[providers.*]` section.
3. Every provider's `cli` binary is present on `PATH`. Missing binary → fail fast with a hint pointing to the role config and the install path for the provider.
4. The PRD parses cleanly under the strict grammar (including resolution of every `**Skills**` reference to a real `.zurdo/skills/<name>/` directory).
5. The PRD's `prd_hash` matches the recorded hash in `prd.json` (if present) — else abort with diff and require `--reset` or `--accept-changes`.
6. If `--create-prs` is passed: `gh` is present on `PATH` and `--no-branch` is not also passed (the two flags are mutually exclusive).

Credentials (env vars like `ANTHROPIC_API_KEY`) are **not** validated. If the binary is present but auth is broken, that surfaces on the first agent invocation as a normal failed iteration with the CLI's own stderr captured.

---

## 5. Cross-Cutting Implications

### 5.1 What Got Cut from the Original Spec

- **GitHub Copilot as a v1 provider.** Cut. Architecture supports it; capability doesn't.
- **Parallel task execution.** Deferred. Sequential only.
- **Auto-merge of dependency branches.** Cut. Dependencies are ordering-only; cumulative branching is the substitute.
- **Reading GFM checkbox state.** Cut. Manual criteria carry no machine signal.
- **Per-PRD config overrides.** Deferred.
- **User-configurable prompt template.** Deferred.
- **Per-criterion timeout override syntax.** Deferred; global default only.
- **`**Working-dir**` task field.** Cut; users compose `cd` in `[shell:]`.
- **Skill inlining into the prompt.** Cut. Skills are installed natively by the provider adapter or silently ignored; no text-paste fallback. See §3.6.
- **`metrics.jsonl` streaming.** Cut from v1. `prd.json` plus the curated report schema (§5.2) cover v1 observability needs. Cross-run analytics are a non-goal for v1; reintroduce when a real consumer appears.
- **State schema migration framework.** Deferred to v2. `schema_version` is recorded in `prd.json` but any mismatch is a fatal error directing the user to `--reset`.
- **Non-GitHub VCS hosts.** Deferred. v1 ships only `GitHubAdapter` (using `gh`) for `--create-prs`.

### 5.2 What Got Added Beyond the Original Spec

- **`passed-pending-review` task status.** Distinguishes "automated work done, human review outstanding" from plain `passed`.
- **`blocked-by-dependency` as a terminal status** that propagates through the dep graph.
- **Pre-flight criteria check (iteration 0)** that can mark a task `passed` without invoking the LLM at all.
- **Per-iteration commits with structured trailers**, including failed iterations.
- **Cumulative/stacked branching** as the answer to the "Dependency Branch Merging" open question.
- **`prd_hash` validation on resume** with `--reset` / `--accept-changes` flags.
- **Append-only `iterations` array** in `prd.json` for full audit history.
- **Per-task `**Agent-timeout**`** metadata field (optional, falls back to `timeouts.agent_seconds`).
- **`zurdo init`** subcommand to generate default config.
- **Pre-flight CLI binary validation.**
- **`[defaults]` config section** for task-metadata fallbacks (`max_attempts`, alongside `timeouts.agent_seconds` for `Agent-timeout`). `Max-Attempts` and `Agent-timeout` are now optional on tasks.
- **Skills system with provider-adapter install model.** Skills live at `.zurdo/skills/<name>/SKILL.md` in Anthropic-compatible format; provider adapters install them natively or silently skip (§3.6).
- **`--create-prs` opt-in flag.** PR creation is off by default; passing the flag requires `gh` on `PATH` and triggers per-task PR creation on terminal-pass.
- **`VcsHostAdapter` trait** with narrow surface (`create_pr`, `update_pr_status`). v1 ships only `GitHubAdapter`.
- **Per-task `pr_url` field** in `prd.json`. Persisted on successful PR creation; retried on resume when null and the task is terminal-pass.
- **Rich PR bodies** generated via the `reporter` role: task Description, final criteria results, iteration narrative ("iteration 1 failed on X; iteration 2 fixed by Y"). Cached in `prd.json` to avoid re-invoking the reporter on subsequent runs.
- **`zurdo report` subcommand** with curated report schema (separate from `prd.json`), default JSON output, `--format md` to switch. Auto-written to `.zurdo/<slug>/reports/<timestamp>.<ext>` and echoed to stdout.
- **`report_schema_version`** independent of `prd.json` schema version, evolving separately.
- **Structured logging** to stderr by default, `--log-file <path>` to opt in. `-v` / `-q` shorthand plus `--log-level=<level>` (passing both is an error). `--log-format=<text|json>` orthogonal. Standard error/warn/info/debug/trace ladder.
- **Three-layer test strategy** (unit / integration with `MockAgentCli` / end-to-end real-LLM smoke), all three required for v1. `MockAgentCli` lives in PRD-06. Enumerated test scenarios per PRD plus a 70% soft coverage floor. Layer 3 runs on pre-release tags only.

### 5.3 Resolved Open Questions from Original Spec

- **Provider Abstraction Layer:** two distinct traits (`AgentCli`, `CompletionCli`), not one.
- **Effort Map + Provider Interaction:** effort selects model within provider; provider is fixed by role config.
- **CLI Output Parsing:** per-provider hardcoded parser for completion role; output not parsed at all for agent role.
- **Dependency Branch Merging:** ordering-only deps + cumulative branching; no auto-merge.
- **`--no-branch` semantics:** all git side-effects skipped; `prd.json` state-tracking unchanged.

### 5.4 Resolved Open Items (Post-Interview)

The following items were resolved in the open-questions interview that followed the initial spec-enhancement interview. Each item now has a settled decision; the rationale and cascading consequences are captured in the relevant sections above (§1–§4) and in §5.1 / §5.2.

1. **`Max-Attempts` default — Resolved.** Optional on tasks. Falls back to `[defaults] max_attempts` in `.zurdo/config.toml`. Same treatment applied to `Agent-timeout` (already had `timeouts.agent_seconds` as its default). `Effort` and `Depends-on` remain required on every task; `Effort` has no sensible global default and `Depends-on` is inherently per-task. See §1.1 and §4.2.

2. **Skills format and resolution — Resolved.** Skills live at `.zurdo/skills/<name>/SKILL.md` in Anthropic-compatible format (YAML frontmatter + markdown body + optional sibling files in the same directory). Flat layout; no `global/` subdir in v1. The `AgentCli` adapter for each provider decides how to install skills natively (Anthropic adapter: point Claude at `.zurdo/skills/` via Claude's configuration). Providers without a native skill system return `NotSupported` and skills are silently ignored with a `warn` log. No inlining fallback. Missing skill directory is a parse-time error. See §3.6.

3. **PR creation mechanics — Resolved.** PR creation is opt-in via `--create-prs`. When the flag is set, `gh` becomes a hard pre-flight requirement; `--create-prs` and `--no-branch` are mutually exclusive. v1 defines a narrow `VcsHostAdapter` trait (`create_pr`, `update_pr_status`) and ships only `GitHubAdapter`. PRs are created per task as soon as the task reaches `passed` or `passed-pending-review`, not in batch at run-end. PR bodies are rich: task Description verbatim, final criteria pass/fail, link to per-iteration commits, plus an iteration narrative generated by the `reporter` role and cached in `prd.json` (`pr_url` and `iteration_summary` fields). On PR-creation failure, the task stays terminal-pass and `pr_url` is null; later runs retry. See §4.3 and §5.2.

4. **`--report` output format — Resolved.** Default JSON, `--format md` to switch to markdown. No TTY detection — predictable, scriptable. Reports auto-write to `.zurdo/<slug>/reports/<timestamp>.<ext>` and are also echoed to stdout. Content is a **curated report schema** distinct from `prd.json`, with its own `report_schema_version`; the markdown renderer consumes the same schema, so the two formats stay in sync by construction. See §5.2; full schema design lives in PRD-09.

5. **`metrics.jsonl` schema — Resolved by cutting.** Removed from v1. `prd.json` plus the curated report schema cover v1 observability needs. Cross-run analytics are a non-goal for v1 and will be revisited when a concrete consumer (dashboard, analytics pipeline) appears. See §5.1.

6. **Schema migration story — Resolved by deferring.** No migration framework in v1. `schema_version` is recorded in `prd.json` so a future Zurdo has a clean signal to branch on, but in v1 any mismatch is a fatal error with a clear message directing the user to `--reset`. Migration design begins when v2 ships a concrete schema delta to design against. See §5.1.

7. **Logging — Resolved.** Stderr by default; `--log-file <path>` opts in to a file. `-v` / `-q` shorthand (aliases for `--log-level=debug` / `--log-level=warn`); passing both `-v`/`-q` and `--log-level` is an error rather than a precedence rule. `--log-format=<text|json>` orthogonal. Standard ladder: **error** for fatal conditions, **warn** for soft failures (PR creation failed, skill provider mismatch, criterion timeout), **info** for task lifecycle (default), **debug** for iteration internals, **trace** for everything including full prompts and untruncated criterion output. See §5.2; full CLI surface in PRD-10.

8. **Test strategy — Resolved.** Three layers, all required for v1:
   - **Layer 1: unit tests** for pure logic (parser, topo-sort, state machine, config, prompt rendering, report serialization). No I/O.
   - **Layer 2: integration tests** with a `MockAgentCli` taking scripted invocation sequences. Git, subprocesses, filesystem all real (tmpdir-backed repos); only the agent is faked. Covers the runner loop, cumulative branching, crash recovery, Ctrl-C, iteration limits.
   - **Layer 3: end-to-end smoke tests** against real `claude` in CI, gated to pre-release tags only (`v*-rc*` or equivalent). Locally invoked via a `e2e` feature flag and skipped without `ANTHROPIC_API_KEY`. Two fixture PRDs: a happy-path single-task PRD and an iteration-recovery PRD.

   `MockAgentCli` and shared test infrastructure live in PRD-06 alongside the `AgentCli` trait; PRDs 04, 05, 07 depend on PRD-06 for their integration tests, so PRD-06 must ship its mock infrastructure early (mock first, real Anthropic adapter second). Each PRD enumerates the specific test scenarios it must exercise as part of its acceptance criteria; a 70% line-coverage soft floor reports informationally (warning, not hard fail) and will be revisited once v1 stabilizes. See §5.2.

### 5.5 Revised PRD Partitioning

Given the resolutions above, the partitioning into PRDs:

| PRD                                           | Scope                                                                                                                                                                                                                                                                                               |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **PRD-01: Strict PRD Grammar & Parser**       | The markdown grammar, validation rules, error reporting. Validator takes the **repo root** as input so it can resolve `**Skills**` references against `.zurdo/skills/`.                                                                                                                             |
| **PRD-02: Config & Pre-flight Validation**    | `.zurdo/config.toml` schema (including `[defaults]`), `zurdo init`, pre-flight binary checks, `--create-prs` ⇒ `gh` requirement and `--no-branch` mutual exclusion.                                                                                                                                 |
| **PRD-03: State Management (`prd.json`)**     | Schema, atomic writes, `prd_hash` validation, `--reset` / `--accept-changes`. `schema_version` recorded but no migration framework: mismatch is fatal in v1.                                                                                                                                        |
| **PRD-04: Sequential Task Runner**            | Topo-sort, status state machine, pre-flight criteria check (iteration 0), iteration loop, crash recovery, Ctrl-C handling.                                                                                                                                                                          |
| **PRD-05: Criteria Execution**                | All five hint types, AND'ing, timeouts, working-dir guarantees, branch checkout precondition.                                                                                                                                                                                                       |
| **PRD-06: Provider Layer & Agent Invocation** | `AgentCli` / `CompletionCli` traits, Anthropic implementation, prompt template, effort→model resolution. **Also ships `MockAgentCli` and shared test infrastructure** (mock first, real adapter second).                                                                                            |
| **PRD-07: Git & VCS Operations**              | Cumulative branching, per-iteration commits with trailers, `--no-branch`. `VcsHostAdapter` trait, `GitHubAdapter` (`gh`-based), `--create-prs` flag, per-task PR creation on terminal-pass, rich PR body generation (calls `reporter` role), `pr_url` persistence and retry.                        |
| **PRD-08: Skills System**                     | Skill file format (Anthropic-compatible at `.zurdo/skills/<name>/SKILL.md`), parse-time validation of `**Skills**` references, `install_skills` adapter method, Anthropic-provider installation strategy, warn-and-skip behavior for unsupported providers.                                         |
| **PRD-09: Reporting**                         | Curated report schema (with `report_schema_version`), JSON and markdown serializers from the shared schema, `zurdo report` subcommand, auto-write to `.zurdo/<slug>/reports/`. Iteration-narrative generation via the `reporter` role (shared with PRD-07's PR body content; cached in `prd.json`). |
| **PRD-10: CLI Surface**                       | Subcommands (`init`, `run`, `report`, `migrate` stub, `validate`?), flags (`--create-prs`, `--no-branch`, `--reset`, `--accept-changes`, `--log-file`, `-v`/`-q`, `--log-level`, `--log-format`, `--format`), exit codes, help text, logging plumbing (stderr default, ladder mapping).             |

PRDs 01–06 form the v1 "minimum viable" path. PRDs 07–10 layer on top. Within the v1 core, the dependency order is roughly: 01 → 02 → 03 → 04, with 05 and 06 unlockable in parallel once 04 has its skeleton. PRD-06's mock infrastructure must land before PRDs 04/05/07 can complete their integration tests, so 06's internal ordering is "mock-first."

**Cross-PRD coupling worth flagging:**

- **PRD-07 calls the `reporter` role at PR-creation time** to generate the iteration narrative for the PR body. This is the same narrative consumed by PRD-09 reports. The reporter invocation logic belongs in PRD-09 (or in PRD-06's role-invocation surface); PRD-07 consumes it.
- **PRD-08 (Skills) extends the `AgentCli` trait** with `install_skills`. The trait itself lives in PRD-06; PRD-08 specifies the method signature, the Anthropic implementation, and the `SkillInstallOutcome` semantics. Order: PRD-06 defines the trait shape including `install_skills`, PRD-08 fills in the behavior.
- **PRD-01 (Parser) needs filesystem access** to validate `**Skills**` references. PRD-01's interface takes a repo root, not just a markdown string. This loosens "pure parsing" framing but keeps validation in one place.
