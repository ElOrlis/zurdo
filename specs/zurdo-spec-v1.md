# Zurdo — Final Spec (v1)

> A Rust rewrite of [paullovvik/ralph-loop](https://github.com/paullovvik/ralph-loop).
> Goal: preserve the simplicity of the original CLI while adding targeted improvements.
> Intentionally excludes: GitHub Projects v2, mcpls, sync systems.

This document is the authoritative v1 specification. It consolidates the original feature spec with all decisions made in the spec-enhancement and open-questions interviews. Every previously-open question has been resolved. See §11 for the PRD partitioning that this spec is partitioned into for implementation.

---

## Table of Contents

1. [Overview](#1-overview)
2. [PRD Format & Grammar](#2-prd-format--grammar)
3. [Acceptance Criteria](#3-acceptance-criteria)
4. [Task Status & Dependencies](#4-task-status--dependencies)
5. [Runtime Semantics](#5-runtime-semantics)
6. [Git & VCS Operations](#6-git--vcs-operations)
7. [Provider Layer](#7-provider-layer)
8. [Skills System](#8-skills-system)
9. [PRD Analysis (`--analyze`)](#9-prd-analysis---analyze)
10. [Reporting (`zurdo report`)](#10-reporting-zurdo-report)
11. [Configuration](#11-configuration)
12. [CLI Surface](#12-cli-surface)
13. [Logging](#13-logging)
14. [Test Strategy](#14-test-strategy)
15. [PRD Partitioning](#15-prd-partitioning)
16. [Out of Scope for v1](#16-out-of-scope-for-v1)

---

## 1. Overview

Zurdo runs a PRD end-to-end: it parses a markdown task file, walks the dependency graph in topological order, and for each task it loops an LLM agent against the working tree until acceptance criteria pass or a per-task attempt budget is exhausted. All state lives in `.zurdo/<slug>/` under the repo root. v1 is sequential, supports a single LLM provider (Anthropic via the `claude` CLI), and ships optional GitHub PR creation behind a flag.

The architectural shape:

- **Strict PRD grammar.** PRDs are markdown but constrained to a closed grammar; anything off-grammar is a parse error with a line number.
- **Two provider traits.** `AgentCli` for the executor (mutates the working tree); `CompletionCli` for verifier / analyzer / reporter (returns text).
- **State in `prd.json`.** Single source of truth. Git artifacts are informational. Crash recovery, resume, reporting all read from `prd.json`.
- **Cumulative branching.** Each task branches off the previous terminal-pass task's branch, producing a deterministic stack.
- **Skills installed natively.** Skills live at `.zurdo/skills/<name>/SKILL.md`; provider adapters install them via the provider's own mechanism, or `NotSupported` is logged.

---

## 2. PRD Format & Grammar

### 2.1 File Format

PRDs are **pure markdown with a strict, formally specified grammar**. Anything outside the grammar is a parse error with a precise line number and a suggested correction.

**Multiple PRDs per repo.** One PRD is selected per Zurdo invocation via CLI argument. Each PRD has its own state directory under `.zurdo/<slug>/`, where `<slug>` is derived from the PRD filename plus a short hash for uniqueness:

```
<slug> = <basename>-<hash4>
hash4  = sha1(repo-relative path)[0..4]
```

### 2.2 Grammar

```markdown
# PRD: <free-form title>

<free-form prose intro — not parsed>

## Task: <task-id> — <task title>

**Effort**: <key from effort_map>
**Depends-on**: [<task-id>, <task-id>, ...]
**Max-Attempts**: <integer> # optional
**Skills**: <skill-name>, <skill-name> # optional
**Agent-timeout**: <duration> # optional, e.g. `30m`, `1h`, `45s`

### Description

<free-form prose, passed verbatim to the executor>

### Acceptance Criteria

- [ ] <human-readable criterion text> [<hint>] [<hint>] ...
- [ ] ...
```

### 2.3 Strict Rules

1. Task headings are H2 with the exact prefix `## Task: <id> — <title>`. The em-dash is the separator. `<id>` must match `^task-[a-z0-9-]+$` and be unique within the PRD.
2. Metadata fields are exactly `**Key**: value` on their own line, in a contiguous block under the H2 (no blank lines between them). Keys are a closed enum: `Effort`, `Depends-on`, `Max-Attempts`, `Skills`, `Agent-timeout`. Unknown keys are errors.
3. `Effort` and `Depends-on` are required. `Max-Attempts`, `Skills`, and `Agent-timeout` are optional and fall back to config defaults (see §11.2).
4. `Depends-on` uses YAML-style array syntax: `[task-1, task-2]`. Empty array `[]` for no deps. Cross-PRD dependencies are not allowed.
5. Section headings under each task are H3 from a closed enum: `Description`, `Acceptance Criteria`. Order is enforced.
6. Acceptance criteria are GFM task-list items (`- [ ]`). Trailing `[hint]` blocks are greedily consumed from the end of the line, in source order.
7. Every criterion must have at least one type hint. A criterion with no hint is a validation error. `[manual]` must be explicit if intended.
8. Duration values (`Agent-timeout`) require explicit units (`s`/`m`/`h`). A bare integer is an error.
9. **Effort values are a closed enum defined by `effort_map` in config.** `**Effort**: medium` is valid iff `medium` is a key in `effort_map`. The grammar does not hardcode `low | medium | high`.
10. **Zurdo never modifies the PRD file.** Reading is one-way.

### 2.4 State Directory Layout

```
.zurdo/
├── config.toml
├── skills/
│   └── <skill-name>/
│       ├── SKILL.md
│       └── <optional sibling files>
└── <slug>/
    ├── prd.json
    ├── reports/
    │   └── <timestamp>.<json|md>
    └── .source                       # repo-relative PRD path (collision sentinel)
```

`.zurdo/` should be added to `.gitignore`. Zurdo prints a one-time hint but does not modify `.gitignore`.

---

## 3. Acceptance Criteria

### 3.1 Hint Types

| Hint                                 | Behavior                                              |
| ------------------------------------ | ----------------------------------------------------- |
| `[shell: <cmd>]`                     | Run shell command; pass iff exit code 0.              |
| `[http: <method> <url> -> <status>]` | Make HTTP request; pass iff response status matches.  |
| `[file-exists: <path>]`              | Pass iff file exists at path (relative to repo root). |
| `[grep: <pattern> in <file>]`        | Pass iff pattern is found in file.                    |
| `[manual]`                           | No machine check. Presence affects task status only.  |

### 3.2 Semantics

- **Multiple hints per criterion are AND'd.** All must pass. Hints run and report in source order. Failure messages identify the specific hint that failed.
- **`[manual]` mixed with automated hints:** the manual portion is ignored. Automated hints still gate the criterion. Manual verification is out-of-band; defects found manually go into a new PRD.
- **Manual criteria carry zero machine signal.** Zurdo never reads or writes GFM checkbox state (`- [x]` vs `- [ ]`). Checkboxes are personal bookkeeping for humans.
- Criteria run concurrently within a task via `tokio`.

### 3.3 Execution

- **Working directory:** always at repo root. Users compose `cd subdir && cmd` inside `[shell:]` if a subdirectory is needed. No `**Working-dir**` field.
- **Branch state:** Zurdo ensures the task's branch is checked out before running criteria. Criteria reflect branch state, not main.
- **Timeouts:** global default in `.zurdo/config.toml` under `timeouts.criterion_seconds` (default 300 = 5 minutes). Applies to `shell` and `http` only. `file-exists` and `grep` are fast enough to not need timeouts. `[manual]` doesn't run. No per-criterion timeout override in v1.
- **On timeout:** kill the criterion's process; record as a failed result with `timed_out: true` in the per-criterion result.

---

## 4. Task Status & Dependencies

### 4.1 Status Enum

| Status                  | Meaning                                                                                                       | Terminal? |
| ----------------------- | ------------------------------------------------------------------------------------------------------------- | --------- |
| `pending`               | Not yet attempted.                                                                                            | No        |
| `blocked`               | Waiting on running dependencies.                                                                              | No        |
| `in-progress`           | Currently iterating.                                                                                          | No        |
| `passed`                | All criteria passed; no manual criteria present.                                                              | Yes       |
| `passed-pending-review` | All automated criteria passed; one or more manual criteria are present (human review obligation outstanding). | Yes       |
| `failed`                | Exhausted `Max-Attempts` without satisfying all automated criteria.                                           | Yes       |
| `blocked-by-dependency` | An upstream dependency reached `failed` or `blocked-by-dependency`. Never runs.                               | Yes       |

`blocked-by-dependency` propagates transitively. Reports distinguish failures from blocked-by-dependency tasks so root causes are clear.

### 4.2 Dependency Semantics

- **Syntax:** `**Depends-on**: [task-1, task-2]`. YAML-style array. Empty array for no deps.
- **Scope:** local to one PRD only.
- **Semantics: ordering-only.** A dependency means "this task does not start until its dependencies have reached a terminal-pass state." It does not affect git branch lineage in any logical sense beyond what falls out of cumulative branching (§6.2).
- **A dep is satisfied when the dependency reaches `passed` or `passed-pending-review`.** `passed-pending-review` is sufficient because manual review is out-of-band.
- Cycles, self-dependencies, and dangling references are caught at validation.

---

## 5. Runtime Semantics

### 5.1 Concurrency

**Sequential execution only in v1.** One task at a time. Parallel execution (`--parallel N`) is a deferred future feature.

**Ordering:** topo-sort the dep graph; among runnable tasks (deps all satisfied, not blocked-by-dependency), pick the one that appeared earliest in the PRD's source order. Declaration order is a deterministic tiebreaker.

### 5.2 State Persistence

**`prd.json` is the single source of truth.** Git artifacts are informational only. Crash recovery, resume, and reporting all read from `prd.json`. Atomic writes via temp-file-and-rename.

Schema (v1):

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
        }
      ]
    }
  }
}
```

**Notes:**

- Per-criterion `stdout_truncated` / `stderr_truncated` are truncated to 4KB each (tail preserved).
- The `iterations` array is append-only — every attempt is retained for reporting.
- `pr_url` populated after successful PR creation (only when `--create-prs` was passed and the task reached a terminal-pass state). Null otherwise. On resume, any terminal-pass task with `pr_url: null` triggers a PR-creation retry.
- `iteration_summary` is generated by the `reporter` role at PR-creation time (and at `zurdo report` time when missing) and cached so the reporter is not re-invoked on subsequent runs.
- **No `metrics.jsonl` in v1.** `prd.json` plus the curated report schema (§10) cover v1 observability needs.

### 5.3 Per-Iteration Lifecycle

**Pre-flight check (iteration 0).** Before invoking the agent for a task, Zurdo runs all the task's criteria against the current state of the task's branch (or `main` if the branch hasn't been created). If all pass: mark task `passed` with `attempts: 0`, do **not** create a branch or PR, skip to the next task.

**Per-iteration steps** (when at least one criterion fails on pre-flight):

1. Ensure task branch exists. If not, create it off the most recently terminal-passed task's branch in this run (or `main` if none yet — see §6.2).
2. Build the agent prompt (§7.4).
3. Invoke `AgentCli` against the prompt. Bounded by `Agent-timeout` (per-task) or `timeouts.agent_seconds` (default).
4. Commit whatever the agent did on the task's branch with a structured trailer. Failed iterations still commit — full audit trail.
5. Re-run **all** criteria (including ones that passed last iteration — regression detection).
6. Record per-criterion results in `prd.json` under this attempt's entry.
7. Increment `attempts` **only after the commit succeeds**; crashes before this point do not consume budget.
8. If all criteria pass → `passed` (or `passed-pending-review` if manual criteria exist). Else if `attempts >= Max-Attempts` → `failed`. Else → loop.

### 5.4 Failure and Recovery

- **Crash mid-iteration:** next run detects uncommitted changes on the in-progress task's branch → `git reset --hard HEAD` → restart the iteration. `attempts` is **not** incremented (the previous attempt didn't complete a commit).
- **Ctrl-C, first press:** finish the current iteration cleanly (commit, update state, exit).
- **Ctrl-C, second press:** hard exit; rely on crash recovery on next invocation.
- **Agent timeout:** kill the process group (not just the parent PID), `git reset --hard HEAD`, count as a failed attempt (distinct from a "criteria failed" attempt in metrics).
- **Agent exit 0 but criteria fail:** treat as a normal failed iteration. Agent's nonzero exit is informative but not authoritative; criteria are authoritative.
- **PRD hash mismatch at run start:** abort with a structured diff of what changed (tasks added/removed/modified). Two flags to proceed:
  - `--reset`: archive old state, start over.
  - `--accept-changes`: preserve status of unchanged tasks; new/changed tasks become `pending`.
- **Schema version mismatch in `prd.json`:** fatal error with a clear message directing the user to `--reset`. No migration framework in v1; `schema_version` is recorded so a future Zurdo has a clean signal to branch on.

---

## 6. Git & VCS Operations

### 6.1 Branch Naming

```
zurdo/<prd-slug>/<task-id>-<title-slug>
```

### 6.2 Cumulative Branching

When a task creates its branch, it branches off the **most recently terminal-passed task's branch in this run**, not `main`. If no task has reached terminal-pass yet in this run, branch off `main`.

This produces a deterministic, linear stack of branches. Downstream tasks naturally see upstream tasks' code without any merge logic. Reviewers see a stack of PRs each containing only that task's incremental diff, all targeting `main`.

The `--no-branch` flag opts out of **all** git side-effects entirely; `prd.json` state-tracking is unchanged.

### 6.3 Commit Format

```
<task-id>: <title>

Iteration <n>. Criteria: <x>/<y> passing.

Zurdo-Task-Id: <task-id>
Zurdo-Status: in-progress | passed | passed-pending-review | failed
```

Failed iterations still commit so history stays complete.

### 6.4 PR Creation (`--create-prs`)

PR creation is **opt-in via `--create-prs`**. When the flag is set:

- `gh` becomes a hard pre-flight requirement.
- `--create-prs` and `--no-branch` are **mutually exclusive**.
- PRs are created **per task as soon as the task reaches `passed` or `passed-pending-review`**, not in batch at run-end.
- PR bodies are rich: task Description verbatim, final criteria pass/fail summary, link to per-iteration commits, plus an iteration narrative generated by the `reporter` role.
- The iteration narrative is cached in `prd.json` (`iteration_summary` field) so the reporter is not re-invoked on subsequent runs.
- On PR-creation failure, the task stays terminal-pass and `pr_url` stays null. Later runs retry.

v1 defines a narrow `VcsHostAdapter` trait:

```rust
trait VcsHostAdapter {
    fn create_pr(&self, branch: &str, base: &str, title: &str, body: &str) -> Result<String>;
    fn update_pr_status(&self, pr_url: &str, status: PrStatus) -> Result<()>;
}
```

v1 ships only `GitHubAdapter` (using `gh`). Non-GitHub VCS hosts are deferred.

---

## 7. Provider Layer

### 7.1 Two Distinct Traits

Two separate provider traits, deliberately not unified:

```rust
// For the executor role only — agents that modify files.
trait AgentCli {
    fn invoke(&self, prompt: &str, repo_root: &Path) -> Result<AgentRunOutcome>;
    fn install_skills(&self, skills: &[Skill], repo_root: &Path) -> Result<SkillInstallOutcome>;
}

struct AgentRunOutcome {
    exit_code: i32,
    stdout: String,         // captured for logs, not parsed for content
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

### 7.2 V1 Provider Matrix

| Provider             | Implements `AgentCli` | Implements `CompletionCli` |
| -------------------- | --------------------- | -------------------------- |
| Anthropic (`claude`) | Yes                   | Yes                        |
| GitHub Copilot       | **Cut from v1**       | **Cut from v1**            |

GitHub Copilot is cut from v1 because its CLI surface doesn't support the "run an agent against a task" pattern Zurdo needs. The architecture supports additional providers cleanly; adding Codex, Gemini, etc. is a follow-on.

### 7.3 Roles

| Role       | Description                                                        | Trait used      |
| ---------- | ------------------------------------------------------------------ | --------------- |
| `executor` | Drives each task iteration (the main agent).                       | `AgentCli`      |
| `verifier` | Reviews failed criteria and suggests fixes (optional second pass). | `CompletionCli` |
| `analyzer` | Powers `--analyze` PRD quality feedback.                           | `CompletionCli` |
| `reporter` | Powers PR body generation and `zurdo report` summarization.        | `CompletionCli` |

If a role has no explicit config it inherits the `executor` provider as a fallback.

### 7.4 Executor Prompt Template

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

**Skills are installed into the provider's native skill discovery path, not inlined into the prompt.** See §8. The prompt template contains no `# Skills` section.

**Failure stdout/stderr in the prompt** is truncated to 4KB each (same limit as `prd.json` storage). Truncation preserves the tail (most recent output).

### 7.5 Effort → Model Resolution

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

`effort_map` applies only to the executor. Other roles (`verifier`, `analyzer`, `reporter`) use their role config's `model` directly.

### 7.6 Output Parsing

- **`AgentCli`:** output is not parsed for content. Zurdo captures stdout/stderr for logs and reads the exit code. The "real output" of an agent invocation is the modified working tree.
- **`CompletionCli`:** per-provider hardcoded parser in Zurdo's source. No configurable regex, no reliance on CLI flags being right.

---

## 8. Skills System

### 8.1 Storage

Skills live at `.zurdo/skills/<name>/SKILL.md` in **Anthropic-compatible format**: YAML frontmatter (`name`, `description`, optional `allowed-tools`), markdown body, optional sibling files in the same directory.

Flat layout — no `global/` subdir in v1.

### 8.2 Reference

The `**Skills**: <name>, <name>` task metadata field names skills the executor should have available. A reference to a non-existent skill directory is a **parse-time error** caught by the grammar validator. This is why the PRD-01 validator takes the repo root as input, not just a markdown string.

### 8.3 Installation Model

Zurdo hosts skills in its own location; the `AgentCli` adapter for each provider decides how to make them available to the agent.

```rust
enum SkillInstallOutcome {
    Installed,         // skills available to agent via provider's native mechanism
    NotSupported,      // provider has no skill system; skills silently ignored
}
```

**Per-provider behavior in v1:**

| Provider             | `install_skills` behavior                                                                                                                    |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Anthropic (`claude`) | Point Claude at `.zurdo/skills/` via Claude's configuration so it discovers skills natively. No copying, no symlinking. Returns `Installed`. |

### 8.4 Unsupported Providers

When `install_skills` returns `NotSupported`, Zurdo logs a `warn`-level message per skill ("skill `auth` ignored: provider `xyz` does not support skills") and proceeds. The task may fail for lack of the skill's guidance, but that surfaces as a normal task failure.

**No inlining fallback in v1.** Inlining `SKILL.md` contents into the prompt for unsupported providers was cut: it complicates the prompt template, doesn't carry sibling files, and creates two divergent skill-authoring conventions. Skill-using PRDs implicitly require a skill-capable provider.

---

## 9. PRD Analysis (`--analyze`)

`--analyze` runs a comprehensive pre-flight check on a PRD and exits without executing any tasks. It is the dedicated "is this PRD well-formed and runnable?" pass, surfacing structural problems and quality issues in one place before a user commits compute to a real run.

### 9.1 Invocation

```
zurdo --analyze <prd-path>
```

Analyze is standalone — it never proceeds to task execution. To run after analyzing, invoke `zurdo` again without `--analyze`.

### 9.2 Checks

Analyze combines deterministic static checks with LLM-powered quality checks via the `analyzer` role.

| Category                    | Check                                                                                         | Type          |
| --------------------------- | --------------------------------------------------------------------------------------------- | ------------- |
| Required fields             | Every task has `id`, `title`, body, `Effort`, `Depends-on`                                    | Deterministic |
| Dependency graph            | No cycles, no self-deps, no dangling refs                                                     | Deterministic |
| Dependency graph            | Resolved topological execution order                                                          | Deterministic |
| Criteria hints              | Every criterion has a recognized type hint (`shell`, `http`, `file-exists`, `grep`, `manual`) | Deterministic |
| Criteria hints              | Hint syntax parses cleanly                                                                    | Deterministic |
| Effort declarations         | `**Effort**` is one of the keys in `effort_map`                                               | Deterministic |
| Skills references           | Every `**Skills**: <name>` resolves to an existing `.zurdo/skills/<name>/` dir                | Deterministic |
| Acceptance criteria quality | Criteria are concrete and testable (not vague like "works well")                              | LLM           |
| Task scope                  | Tasks aren't oversized, ambiguous, or doing too many things                                   | LLM           |

The `analyzer` role is always invoked if configured. There is no flag to skip the LLM portion — if you don't want LLM passes, don't configure an `analyzer` role.

### 9.3 Severity Tiers

- **error** — the PRD will not run correctly as-is (cycle, dangling dep, missing required field, malformed criterion hint, dangling skill ref).
- **warning** — the PRD will run but something is likely wrong (vague criterion, oversized task).
- **info** — advisory observation (resolved execution order, etc.).

### 9.4 Output

Plain text to stdout, grouped by task and then by category. Global findings appear in a leading section.

```
== Global ==
[info] Execution order: task-1 → task-2 → task-4 → task-3
[error] Cycle detected: task-5 → task-6 → task-5

== task-1: Set up auth ==
[error] Missing required field: Effort
[warning] Criterion is vague and not directly testable: "should handle edge cases gracefully"

== task-2: Add login endpoint ==
[warning] Criterion is vague and not directly testable: "login should feel snappy"
[info] 3 criteria, all with type hints
```

Analyze reports findings — it does not suggest fixes.

### 9.5 Exit Codes

- `0` — no errors (warnings and info findings do not fail).
- non-zero — one or more errors present.

This makes `--analyze` safe to drop into CI as a PRD lint step.

### 9.6 Behavior Notes

- Analyze does not require git, branching, or network access for deterministic checks.
- LLM checks shell out to the `analyzer` provider the same way executor calls do.
- If the `analyzer` binary is missing from `PATH`, analyze fails fast with a clear error before any checks run.
- Skills are not injected during analyze — the analyzer operates on the raw PRD only.

---

## 10. Reporting (`zurdo report`)

### 10.1 Invocation

```
zurdo report <prd-path>                  # default JSON to stdout, also auto-written
zurdo report <prd-path> --format md      # markdown
```

No TTY detection — output is predictable and scriptable. Reports auto-write to `.zurdo/<slug>/reports/<timestamp>.<ext>` and are also echoed to stdout.

### 10.2 Report Content

Content is a **curated report schema** distinct from `prd.json`, with its own `report_schema_version`. The markdown renderer consumes the same schema, so the two formats stay in sync by construction.

The report surfaces:

- Per-task attempt counts, pass rates, and wall-clock duration.
- Criteria hotspots (criteria that failed most across attempts).
- Provider/model usage summary across the run.
- Stalled task detection (tasks that hit max attempts without progress).
- Iteration narratives (generated by the `reporter` role; cached in `prd.json`).
- Distinction between `failed` and `blocked-by-dependency` so root causes are clear.

### 10.3 Reporter Role Interaction

The `reporter` role generates iteration narratives that are used in two places:

1. PR bodies, at PR-creation time (§6.4).
2. The `zurdo report` output.

Both consume the cached `iteration_summary` field in `prd.json` to avoid re-invoking the reporter.

---

## 11. Configuration

### 11.1 Scope

**Per-repo only in v1.** `.zurdo/config.toml` at the repo root. No user-level config, no per-PRD overrides. If shared defaults across repos are needed, the user manages that out-of-band.

**Config is required.** A missing `.zurdo/config.toml` is an error. Zurdo provides `zurdo init` to write a sensible default with comments — the same file works as documentation.

### 11.2 Config File

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

### 11.3 Pre-flight Validation

At run start, Zurdo validates:

1. `.zurdo/config.toml` exists and parses.
2. Every role's `provider` exists in the `[providers.*]` section.
3. Every provider's `cli` binary is present on `PATH`. Missing binary → fail fast with a hint pointing to the role config and the install path for the provider.
4. The PRD parses cleanly under the strict grammar (including resolution of every `**Skills**` reference to a real `.zurdo/skills/<name>/` directory).
5. The PRD's `prd_hash` matches the recorded hash in `prd.json` (if present) — else abort with diff and require `--reset` or `--accept-changes`.
6. If `--create-prs` is passed: `gh` is present on `PATH` and `--no-branch` is not also passed.

Credentials (env vars like `ANTHROPIC_API_KEY`) are **not** validated. If the binary is present but auth is broken, that surfaces on the first agent invocation as a normal failed iteration with the CLI's own stderr captured.

---

## 12. CLI Surface

### 12.1 Subcommands

| Subcommand             | Purpose                                                                  |
| ---------------------- | ------------------------------------------------------------------------ |
| `zurdo init`           | Write a default `.zurdo/config.toml` with comments.                      |
| `zurdo run <prd>`      | Run the PRD (default if no subcommand and a positional PRD is given).    |
| `zurdo report <prd>`   | Generate a report from `prd.json`.                                       |
| `zurdo validate <prd>` | Run deterministic grammar/structural checks only (no LLM, no execution). |
| `zurdo migrate`        | Stub for future schema migration; v1 always errors with "no migrations". |

### 12.2 Flags

| Flag                        | Effect                                                                                       |
| --------------------------- | -------------------------------------------------------------------------------------------- |
| `--analyze`                 | Run full pre-flight (deterministic + LLM) and exit. See §9.                                  |
| `--create-prs`              | Enable per-task PR creation. Requires `gh` on `PATH`. Mutually exclusive with `--no-branch`. |
| `--no-branch`               | Skip all git side-effects. State-tracking unchanged.                                         |
| `--reset`                   | Archive old state and start over (on `prd_hash` or `schema_version` mismatch).               |
| `--accept-changes`          | Preserve status of unchanged tasks; new/changed tasks become `pending`.                      |
| `--format <json\|md>`       | Output format for `zurdo report`. Default `json`.                                            |
| `--log-file <path>`         | Tee logs to a file in addition to stderr.                                                    |
| `-v` / `-q`                 | Aliases for `--log-level=debug` / `--log-level=warn`. Mutually exclusive with `--log-level`. |
| `--log-level=<level>`       | One of `error`, `warn`, `info`, `debug`, `trace`. Default `info`.                            |
| `--log-format=<text\|json>` | Default `text`.                                                                              |

### 12.3 Exit Codes

| Code | Meaning                                                          |
| ---- | ---------------------------------------------------------------- |
| `0`  | Success.                                                         |
| `1`  | General failure (any unhandled error).                           |
| `2`  | PRD parse / validation error (analyze findings, grammar errors). |
| `3`  | Pre-flight failure (missing config, missing binary).             |
| `4`  | State mismatch requiring `--reset` or `--accept-changes`.        |
| `5`  | One or more tasks finished `failed` or `blocked-by-dependency`.  |

---

## 13. Logging

- **Default sink:** stderr.
- **`--log-file <path>`:** opts in to file tee.
- **`-v` / `-q`:** shorthand for `--log-level=debug` / `--log-level=warn`. Passing both `-v`/`-q` and `--log-level` is an error rather than a precedence rule.
- **`--log-format=<text|json>`:** orthogonal to level.

Standard ladder:

| Level   | What goes there                                                                 |
| ------- | ------------------------------------------------------------------------------- |
| `error` | Fatal conditions (missing binary, unparseable config, unrecoverable git state). |
| `warn`  | Soft failures (PR creation failed, skill provider mismatch, criterion timeout). |
| `info`  | Task lifecycle (default; what the user expects to see in a normal run).         |
| `debug` | Iteration internals (per-iteration commit SHAs, criteria timings).              |
| `trace` | Everything, including full prompts and untruncated criterion output.            |

---

## 14. Test Strategy

Three layers, all required for v1.

### 14.1 Layer 1: Unit Tests

Pure logic, no I/O:

- Parser
- Topo-sort
- Status state machine
- Config loading
- Prompt rendering
- Report serialization (from the curated schema)

### 14.2 Layer 2: Integration Tests with `MockAgentCli`

Scripted invocation sequences via a `MockAgentCli`. Git, subprocesses, filesystem are all real (tmpdir-backed repos); only the agent is faked. Covers:

- Runner loop end-to-end
- Cumulative branching
- Crash recovery
- Ctrl-C handling (first and second press)
- Iteration limits
- Pre-flight criteria pass (iteration 0)
- `prd_hash` mismatch and `--reset` / `--accept-changes`
- PR creation retry on resume

`MockAgentCli` and shared test infrastructure live in PRD-06 alongside the `AgentCli` trait. PRDs 04, 05, 07 depend on PRD-06 for their integration tests, so PRD-06 must ship its mock infrastructure early (**mock first, real Anthropic adapter second**).

### 14.3 Layer 3: End-to-End Smoke Tests

Real `claude` in CI, gated to pre-release tags only (`v*-rc*` or equivalent). Locally invoked via an `e2e` feature flag, skipped without `ANTHROPIC_API_KEY`. Two fixture PRDs minimum:

- A happy-path single-task PRD.
- An iteration-recovery PRD (failure on iteration 1, success on iteration 2).

### 14.4 Coverage

70% line-coverage soft floor reports informationally (warning, not hard fail). Revisited once v1 stabilizes.

---

## 15. PRD Partitioning

The v1 implementation is partitioned into ten PRDs:

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
| **PRD-10: CLI Surface**                       | Subcommands (`init`, `run`, `report`, `validate`, `migrate` stub), flags, exit codes, help text, logging plumbing (stderr default, ladder mapping).                                                                                                                                                 |

### 15.1 Ordering

PRDs 01–06 form the v1 "minimum viable" path. PRDs 07–10 layer on top.

Within the v1 core, the dependency order is roughly: 01 → 02 → 03 → 04, with 05 and 06 unlockable in parallel once 04 has its skeleton. PRD-06's mock infrastructure must land before PRDs 04/05/07 can complete their integration tests, so 06's internal ordering is **mock-first**.

### 15.2 Cross-PRD Coupling

- **PRD-07 calls the `reporter` role at PR-creation time** to generate the iteration narrative for the PR body. This is the same narrative consumed by PRD-09 reports. The reporter invocation logic belongs in PRD-09 (or in PRD-06's role-invocation surface); PRD-07 consumes it.
- **PRD-08 extends the `AgentCli` trait** with `install_skills`. The trait itself lives in PRD-06; PRD-08 specifies the method signature, the Anthropic implementation, and the `SkillInstallOutcome` semantics. Order: PRD-06 defines the trait shape including `install_skills`, PRD-08 fills in the behavior.
- **PRD-01 needs filesystem access** to validate `**Skills**` references. PRD-01's interface takes a repo root, not just a markdown string. This loosens "pure parsing" framing but keeps validation in one place.

---

## 16. Out of Scope for v1

| Item                                           | Disposition | Reason                                                                           |
| ---------------------------------------------- | ----------- | -------------------------------------------------------------------------------- |
| GitHub Projects v2                             | Cut         | Adds `gh` dependency throughout, significant complexity.                         |
| `mcpls` / LSP integration                      | Cut         | Experimental, obscure dependency.                                                |
| GitHub sync system                             | Cut         | Defeats the simplicity goal.                                                     |
| Copilot agent backend                          | Cut         | Out of scope for a focused CLI.                                                  |
| GitHub Copilot as a v1 provider                | Cut         | Architecture supports it; capability doesn't.                                    |
| Parallel task execution                        | Deferred    | Sequential only in v1. `--parallel N` is a future feature.                       |
| Auto-merge of dependency branches              | Cut         | Dependencies are ordering-only; cumulative branching is the substitute.          |
| Reading GFM checkbox state                     | Cut         | Manual criteria carry no machine signal; Zurdo never modifies the PRD.           |
| Per-PRD config overrides                       | Deferred    | One repo-level config in v1.                                                     |
| User-configurable prompt template              | Deferred    | Fixed template in v1.                                                            |
| Per-criterion timeout override syntax          | Deferred    | Global default only.                                                             |
| `**Working-dir**` task field                   | Cut         | Users compose `cd` in `[shell:]`.                                                |
| Skill inlining into the prompt                 | Cut         | Native install only; unsupported providers get a warning.                        |
| `metrics.jsonl` streaming                      | Cut from v1 | `prd.json` + curated report schema cover v1 needs. Revisit with a real consumer. |
| State schema migration framework               | Deferred    | `schema_version` recorded; mismatch fatal in v1.                                 |
| Non-GitHub VCS hosts                           | Deferred    | v1 ships only `GitHubAdapter`.                                                   |
| OpenTelemetry trace export                     | Deferred    | Future addition.                                                                 |
| Retry escalation (effort bump on failure)      | Cut         | Powerful heuristic, adds complexity; opt-in flag if revisited.                   |
| Auto-inferred effort from criterion count/type | Cut         | Explicit declaration only.                                                       |
