# Zurdo — Final Spec (v1)

> A Rust rewrite of [paullovvik/ralph-loop](https://github.com/paullovvik/ralph-loop).
> Goal: preserve the simplicity of the original CLI while adding **independent verification** as the headline improvement.
> Intentionally excludes: GitHub Projects v2, mcpls, sync systems, **all git/VCS automation**, **PR creation**.

This document is the authoritative v1 specification. It consolidates the original feature spec with all decisions made in the spec-enhancement, open-questions, and v1-tightening interviews. Every previously-open question has been resolved. See §15 for the PRD partitioning that this spec is partitioned into for implementation.

---

## Table of Contents

1. [Overview](#1-overview)
2. [PRD Format & Grammar](#2-prd-format--grammar)
3. [Acceptance Criteria](#3-acceptance-criteria)
4. [Task Status & Dependencies](#4-task-status--dependencies)
5. [Runtime Semantics](#5-runtime-semantics)
6. [Progress Log & Resume](#6-progress-log--resume)
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

Zurdo runs a PRD end-to-end: it parses a markdown task file, walks the dependency graph in topological order, and for each task it loops an LLM agent against the working tree until acceptance criteria pass or a per-task attempt budget is exhausted. All state lives in `.zurdo/<slug>/` under the repo root. v1 is sequential, supports two LLM providers (Anthropic via the `claude` CLI and OpenAI via the `codex` CLI — either can fill either role), and **performs no git operations** — the user manages their own VCS.

The architectural shape:

- **Strict PRD grammar.** PRDs are markdown but constrained to a closed grammar; anything off-grammar is a parse error with a line number.
- **Independent verification.** Every acceptance criterion carries machine-checkable type hints (`shell`, `http`, `file-exists`, `grep`, `manual`). Zurdo — not the agent — runs the checks and decides pass/fail. This is the central improvement over ralph-loop's agent-self-reports-completion model.
- **Two provider traits.** `AgentCli` for the executor (mutates the working tree); `CompletionCli` for the `analyzer` role (returns text).
- **State across two files.** `prd.json` is the terminal-state source of truth (task statuses, attempt counts, criteria results per attempt). `progress.log` is the append-only structured event stream used to reconstruct the in-flight state of a crashed run. Resume reads both.
- **No git.** No branches, no commits, no PRs. The agent modifies the working tree; criteria run against it. Whether to commit, branch, or PR the result is the user's call.
- **Skills installed natively.** Skills authored in `.zurdo/skills/<name>/SKILL.md` are copied into the provider's native discovery path (`.claude/skills/<name>/` for Anthropic, `.codex/skills/<name>/` for Codex). `zurdo init` performs the bulk install; the runtime pre-flight auto-installs any referenced skill whose destination is missing, so adding a new skill or PRD reference between inits doesn't require manual re-sync. `zurdo init --sync` re-syncs without rewriting config.

---

## 2. PRD Format & Grammar

### 2.1 File Format

PRDs are **pure markdown with a strict, formally specified grammar**. Anything outside the grammar is a parse error with a precise line number and a suggested correction.

**Multiple PRDs per repo.** One PRD is selected per Zurdo invocation via CLI argument. Each PRD has its own state directory under `.zurdo/<slug>/`, where `<slug>` is derived from the PRD filename plus a short hash for uniqueness:

```
<slug> = <basename>-<hash4>
hash4  = sha1(repo-relative path)[0..4]
```

Collisions are detected by comparing the invoked PRD's repo-relative path against `prd_path` recorded inside `prd.json`. On mismatch, Zurdo errors with a "slug collision detected — rename one PRD" message; no automatic disambiguation in v1.

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
**Category**: <freeform string> # optional, used for report grouping and analyze stats (§9, §10)

### Description

<free-form prose, passed verbatim to the executor>

### Acceptance Criteria

- [ ] <human-readable criterion text> [<hint>] [<hint>] ...
- [ ] ...
```

### 2.3 Strict Rules

1. Task headings are H2 with the exact prefix `## Task: <id> — <title>`. The em-dash is the separator. `<id>` must match `^task-[a-z0-9-]+$` and be unique within the PRD.
2. Metadata fields are exactly `**Key**: value` on their own line, in a contiguous block under the H2 (no blank lines between them). Keys are a closed enum: `Effort`, `Depends-on`, `Max-Attempts`, `Skills`, `Agent-timeout`, `Category`. Unknown keys are errors.
3. `Effort` and `Depends-on` are required. `Max-Attempts`, `Skills`, `Agent-timeout`, and `Category` are optional. The first three fall back to config defaults (see §11.2); `Category` has no default — when absent, the task is grouped as `Uncategorized` in report and analyze stats output.
4. `Depends-on` uses YAML-style array syntax: `[task-1, task-2]`. Empty array `[]` for no deps. Cross-PRD dependencies are not allowed.
5. Section headings under each task are H3 from a closed enum: `Description`, `Acceptance Criteria`. Order is enforced.
6. Acceptance criteria are GFM task-list items (`- [ ]`). Trailing `[hint]` blocks are greedily consumed from the end of the line, in source order.
7. Every criterion must have at least one type hint. A criterion with no hint is a validation error. `[manual]` must be explicit if intended.
8. Duration values (`Agent-timeout`) require explicit units (`s`/`m`/`h`). A bare integer is an error.
9. **Effort values are a closed enum defined by the active executor's `[effort_map.<provider>]` in config.** `**Effort**: medium` is valid iff `medium` is a key in `[effort_map.<roles.executor.provider>]`. The grammar does not hardcode `low | medium | high`.
10. **Zurdo never modifies the PRD file during `zurdo run`.** Reading is one-way; `prd_hash` integrity (§5.5) depends on it. The `--analyze --fix` mode (§9.7) is the sole carve-out: it writes proposals to a sibling `<prd>.proposed.md` and only overwrites the original on explicit user `y` confirmation. Analyze is standalone (§9.1) and never proceeds to a run, so no `prd.json` state is in flight when the carve-out applies.

### 2.4 State Directory Layout

```
.zurdo/
├── config.toml
├── skills/
│   └── <skill-name>/
│       ├── SKILL.md
│       └── <optional sibling files>
└── <slug>/
    ├── prd.json                 # terminal state, atomically updated
    ├── progress.log             # append-only JSONL event stream
    ├── lock                     # process-level lock (pid + start time)
    ├── iterations/
    │   ├── <task-id>-<attempt>.out   # full agent stdout for this iteration
    │   └── <task-id>-<attempt>.err   # full agent stderr for this iteration
    └── reports/
        └── <timestamp>.<json|md>
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
- **All-manual tasks** (every criterion has only `[manual]`): short-circuit at pre-flight to `passed-pending-review` with `attempts: 0`. The agent is never invoked.
- **Manual criteria carry zero machine signal.** Zurdo never reads or writes GFM checkbox state (`- [x]` vs `- [ ]`). Checkboxes are personal bookkeeping for humans.
- Criteria run concurrently within a task via `tokio`.

### 3.3 Execution

- **Working directory:** always at repo root. Users compose `cd subdir && cmd` inside `[shell:]` if a subdirectory is needed. No `**Working-dir**` field.
- **Working-tree state:** criteria run against the current working tree as-is. Zurdo does not check out, reset, or otherwise mutate VCS state.
- **Timeouts:** global default in `.zurdo/config.toml` under `timeouts.criterion_seconds` (default 300 = 5 minutes). Applies to `shell` and `http` only. `file-exists` and `grep` are fast enough to not need timeouts. `[manual]` doesn't run. No per-criterion timeout override in v1.
- **On timeout:** kill the criterion's process group; record as a failed result with `timed_out: true` in the per-criterion result.

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
- **Semantics: ordering-only.** A dependency means "this task does not start until its dependencies have reached a terminal-pass state."
- **A dep is satisfied when the dependency reaches `passed` or `passed-pending-review`.** `passed-pending-review` is sufficient because manual review is out-of-band.
- Cycles, self-dependencies, and dangling references are caught at validation.

---

## 5. Runtime Semantics

### 5.1 Concurrency

**Sequential execution only in v1.** One task at a time. Parallel execution (`--parallel N`) is a deferred future feature.

**Ordering:** topo-sort the dep graph; among runnable tasks (deps all satisfied, not blocked-by-dependency), pick the one that appeared earliest in the PRD's source order. Declaration order is a deterministic tiebreaker.

**Single-process invariant.** At run start, Zurdo acquires `.zurdo/<slug>/lock` (file containing pid + ISO-8601 start time). If the lock is held and the recorded pid is still alive, abort with "another zurdo run is active." If the recorded pid is dead, treat the lock as stale, log a `warn`, overwrite, and proceed.

### 5.2 State Persistence

**`prd.json` is the terminal-state source of truth.** Per-task status, attempt counts, and per-attempt criteria results live here. Atomic writes via temp-file-and-rename.

**`progress.log` is the in-flight event stream.** Append-only JSONL. Used alongside `prd.json` to reconstruct what was happening at the moment of a crash. See §6.

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
      "passed_at": "2026-05-08T10:10:00Z",
      "iterations": [
        {
          "attempt": 1,
          "started_at": "...",
          "ended_at": "...",
          "model": "claude-sonnet-4-7",
          "agent_exit_code": 0,
          "agent_stdout_path": ".zurdo/<slug>/iterations/task-1-1.out",
          "agent_stderr_path": ".zurdo/<slug>/iterations/task-1-1.err",
          "agent_stdout_bytes": 18432,
          "agent_stderr_bytes": 217,
          "tokens_in": 3421,
          "tokens_out": 8209,
          "cost_usd_est": 0.12,
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
- The full agent stdout/stderr per iteration live in adjacent files under `.zurdo/<slug>/iterations/` (untruncated); `prd.json` only carries paths and byte counts.
- The `iterations` array is append-only — every attempt is retained for reporting.
- `model` records the resolved executor model for this attempt (per §7.5 effort → model).
- `tokens_in` / `tokens_out` are extracted from the provider CLI's structured output (per-adapter parser, §7.7). Both are `null` when the adapter could not extract them or the provider's output didn't include token counts.
- `cost_usd_est` is computed as `tokens_in × $/MTok_in + tokens_out × $/MTok_out` using the hardcoded pricing table (§7.8). `null` when the model is absent from the table or when either token count is `null`.
- No `pr_url`, `iteration_summary`, or git fields in v1. Iteration narratives in §10 reports are rendered deterministically from `iterations` and may surface the iteration-file tails.

### 5.3 Per-Iteration Lifecycle

**Pre-flight check (iteration 0).** Before invoking the agent for a task, Zurdo runs all the task's criteria against the current working tree. If all pass: mark task `passed` (or `passed-pending-review` if any `[manual]` criteria are present) with `attempts: 0`, skip to the next task.

**Per-iteration steps** (when at least one criterion fails on pre-flight):

1. Write an `iteration_start` event to `progress.log`.
2. Build the agent prompt (§7.4). On iteration ≥ 2, the prompt includes the tail of the prior iteration's `.out` file (see §7.4).
3. Invoke `AgentCli` against the prompt. Bounded by `Agent-timeout` (per-task) or `timeouts.agent_seconds` (default).
4. Write full agent stdout/stderr to `.zurdo/<slug>/iterations/<task-id>-<attempt>.{out,err}`.
5. Write an `agent_completed` event to `progress.log` capturing exit code, duration, file paths, and byte counts.
6. Re-run **all** criteria (including ones that passed last iteration — regression detection).
7. Write `criterion_result` events per check.
8. Record per-criterion results plus the iteration-file paths/byte counts in `prd.json` under this attempt's entry. Increment `attempts`.
9. Write a `task_status` event to `progress.log` with the new status.
10. If all criteria pass → `passed` (or `passed-pending-review` if manual criteria exist). Else if `attempts >= Max-Attempts` → `failed`. Else → loop.

**Attempt accounting:** an attempt counts only once step 8 (the `prd.json` write) completes. Crashes before that point do not consume budget; resume picks up by re-running the iteration. Orphan iteration files from a crashed attempt are left in place and overwritten if the same `attempt` number is re-tried (which it will be, since `attempts` wasn't incremented).

### 5.4 Transient Agent Errors

Some agent invocation failures are transient (HTTP 429, network timeouts, DNS failures) and should not consume an attempt. The provider adapter classifies its exit/stderr:

- **`Transient`** — retry with exponential backoff: 5s, 10s, 20s, then give up. Does not consume an attempt.
- **`Permanent`** — count as a failed attempt with no retry. The criteria-failure path takes over.

For Anthropic (`claude`), the adapter greps stderr for `rate limit`, `429`, `quota`, `network`, `connection`, `timeout`, `refused` to classify (mirrors ralph-loop's heuristic). All other non-zero exits are `Permanent`.

### 5.5 Failure and Recovery

- **Crash mid-iteration.** On next run, scan `progress.log` for an unbalanced `iteration_start` (no matching `agent_completed`) and treat that attempt as not-counted. The working tree carries whatever the crashed iteration left behind; the next iteration starts from there. **Zurdo does not reset the working tree** — that responsibility is the user's.
- **Ctrl-C, first press:** finish the current iteration cleanly (`agent_completed`, `task_status`, `prd.json` write, exit).
- **Ctrl-C, second press:** hard exit; rely on crash recovery on next invocation.
- **Agent timeout:** kill the process group (not just the parent PID), count as a failed attempt with `timed_out: true` recorded in the iteration entry.
- **Agent exit 0 but criteria fail:** treat as a normal failed iteration. Agent's exit code is informative but not authoritative; criteria are authoritative.
- **PRD hash mismatch at run start:** abort with a structured diff of what changed (tasks added/removed/modified). The only path forward is `--reset` (or the `[X] Reset` choice in the interactive prompt, §6.2). Surgical preservation of unchanged tasks is deferred (see §16).
- **Schema version mismatch in `prd.json`:** fatal error with a clear message directing the user to `--reset`. No migration framework in v1; `schema_version` is recorded so a future Zurdo has a clean signal to branch on.
- **`--reset` semantics:** Zurdo moves the entire existing `.zurdo/<slug>/` directory (except `lock`, which is recreated) into `.zurdo/<slug>/.archive/<UTC-timestamp>/`, then starts a new run with a fresh `prd.json` and `progress.log`. The `iterations/` and `reports/` subtrees are archived alongside so prior runs remain auditable. Archived state is preserved indefinitely; users may delete `.zurdo/<slug>/.archive/` at any time.

### 5.6 Global Iteration Budget (Optional)

Per-task `Max-Attempts` (§2.3.3, §11.2) caps attempts within a single task. A run can still consume up to `Σ tasks × Max-Attempts` total agent invocations if every task hits its individual cap. The optional **global iteration budget** is an aggregate ceiling across the whole run — a safety hatch against runaway compute, not a normal flow.

- **Knob.** Config: `[defaults] max_total_iterations = N`. Flag: `--max-iterations N` (overrides config). Default is `0`, meaning **unlimited**. The budget is opt-in.
- **What counts.** Only attempts that incremented `attempts` in `prd.json` — i.e., real iterations per §5.3 step 8. Transient retries (§5.4), pre-flight short-circuits (§5.3 iteration 0), and crashed iterations whose `prd.json` write never landed (§5.3 attempt accounting) do **not** consume budget.
- **Trip behavior.** Identical to first-press Ctrl-C (§5.5). The currently-running iteration completes cleanly (`agent_completed`, `task_status`, `prd.json` write), then no further iterations or tasks are started. Tasks that never ran retain their current status (typically `pending`).
- **Exit code.** `6` — iteration budget exhausted (see §12.3). Distinct from `5` (task `failed`) so CI and wrappers can branch on it.
- **Surfacing.** The summary table (§13.1.5) renders untouched tasks with `—` in `wall-clock` and `criteria`. The sticky dashboard header (§13.1.6) shows `iter <used>/<cap>` during the run. The next-step suggestion (§13.1.7) recommends re-running with a higher cap.
- **Interaction with `--no-prompt` and resume.** A resumed run shares the budget pool: `used` is the count of iterations executed in *this* invocation, not the lifetime total. Tasks completed in prior runs do not count.

---

## 6. Progress Log & Resume

### 6.1 Format

`progress.log` is append-only JSONL. Each line is one event object. Writes are O_APPEND-flushed per event so a crash truncates at most a partial trailing line.

```jsonl
{"ts":"2026-05-08T10:00:00Z","event":"run_start","prd_hash":"abc...","resumed":false}
{"ts":"2026-05-08T10:00:01Z","event":"iteration_start","task_id":"task-1","attempt":1}
{"ts":"2026-05-08T10:00:02Z","event":"agent_invoked","task_id":"task-1","attempt":1,"provider":"anthropic","model":"claude-sonnet-4-7"}
{"ts":"2026-05-08T10:03:14Z","event":"agent_completed","task_id":"task-1","attempt":1,"exit_code":0,"duration_ms":192100,"stdout_path":".zurdo/<slug>/iterations/task-1-1.out","stderr_path":".zurdo/<slug>/iterations/task-1-1.err","stdout_bytes":18432,"stderr_bytes":217}
{"ts":"2026-05-08T10:03:15Z","event":"criterion_result","task_id":"task-1","attempt":1,"hint":"shell: cargo test","passed":true,"duration_ms":1100}
{"ts":"2026-05-08T10:03:16Z","event":"task_status","task_id":"task-1","attempt":1,"status":"passed","attempts":1}
{"ts":"2026-05-08T10:03:16Z","event":"run_complete","tasks_passed":1,"tasks_failed":0,"tasks_blocked":0}
```

**Event types (closed enum):** `run_start`, `run_complete`, `iteration_start`, `agent_invoked`, `agent_completed`, `criterion_result`, `task_status`, `transient_retry`, `lock_stale`.

**Match field convention.** Events that belong to an iteration always carry both `task_id` and `attempt` (the integer iteration index). `task_status` additionally carries `attempts` (the running total after this attempt) for at-a-glance reporting; matching against `iteration_start` is by `task_id` + `attempt` only.

### 6.2 Resume Reconciliation

On `zurdo run` against an existing `.zurdo/<slug>/prd.json`:

0. **Interactive resume prompt.** If none of `--resume`, `--reset`, or `--no-prompt` was provided AND stdin is a TTY, prompt the user:

   ```
   Existing state found at .zurdo/auth-a1b2/
     Last updated: 2026-05-08T10:15:00Z
     Progress:    3 passed, 1 passed-pending-review, 1 failed, 2 pending
     Last task:   task-rate-limit (failed after 5 attempts)

   What would you like to do?
     [R] Resume from current state            (default)
     [X] Reset — archive state and start over
     [A] Abort
   Choice [R/X/A]:
   ```

   - Default (Enter): `R`.
   - `R`: proceed with the reconciliation steps below.
   - `X`: equivalent to `--reset` — archives `.zurdo/<slug>/` per §5.5 and starts fresh.
   - `A`: exit cleanly with code `0`.
   - **Bypass rules.** Skip the prompt entirely (and behave as `R` — implicit resume) when any of: stdin is not a TTY, `--no-prompt` was passed, `--resume` was passed, `--reset` was passed (behaves as `X`). CI-safe by default.

1. Validate `prd_hash` (§5.5). If mismatch, refuse without `--reset` (or the `[X]` choice in step 0).
2. Read `prd.json` for terminal task statuses (`passed`, `passed-pending-review`, `failed`, `blocked-by-dependency`). These are frozen.
3. Read `progress.log` to identify the most recent in-flight iteration: any `iteration_start` with no `task_status` event sharing the same `task_id` and `attempt`.
4. If found, that attempt did not complete. Discard it (do not increment `attempts`), log a `warn`, and let the runner re-enter that task as the next runnable item.
5. Emit a `run_start` event with `resumed: true`.

Resume requires no git operations. The working tree's current state is taken as given.

### 6.3 No-VCS Posture

v1 ships zero git integration. This is a deliberate scope choice:

- The agent edits files in place. The user reviews `git diff` (or equivalent) when they want.
- There are no Zurdo-created branches, commits, or PRs.
- Resume reads `prd.json` + `progress.log`. Crash recovery does not depend on VCS.
- Users who want per-task commits can run `zurdo run` inside their own scripted workflow that commits after each successful task — Zurdo exits 0 between tasks via `prd.json` updates that the wrapper can observe.

Git/PR automation is reconsidered for v1.x with a concrete UX once the no-VCS baseline is in use.

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
    classification: Classification, // Transient | Permanent | Success
    model: String,          // the resolved model that was actually invoked (§7.5)
    tokens_in: Option<u64>, // extracted from provider output; None if unavailable (§7.7)
    tokens_out: Option<u64>,
}

// For the analyzer role — text completion.
trait CompletionCli {
    fn complete(&self, prompt: &str) -> Result<CompletionOutcome>;
}

struct CompletionOutcome {
    text: String,           // ready-to-use completion content
    raw_stdout: String,     // for debugging
    duration_ms: u64,
    model: String,
    tokens_in: Option<u64>,
    tokens_out: Option<u64>,
}
```

A provider may implement either, both, or neither.

### 7.2 V1 Provider Matrix

| Provider             | Implements `AgentCli` | Implements `CompletionCli` |
| -------------------- | --------------------- | -------------------------- |
| Anthropic (`claude`) | Yes                   | Yes                        |
| OpenAI (`codex`)     | Yes                   | Yes                        |

Either provider can fill either role. `[roles.executor].provider` and `[roles.analyzer].provider` are chosen independently, so a project can run, for example, executor=`codex` + analyzer=`anthropic`.

GitHub Copilot CLI (distinct from OpenAI's Codex CLI) is cut from v1 because its CLI surface doesn't support the "run an agent against a task" pattern Zurdo needs. Gemini and other providers are follow-ons.

### 7.3 Roles

| Role       | Description                                  | Trait used      |
| ---------- | -------------------------------------------- | --------------- |
| `executor` | Drives each task iteration (the main agent). | `AgentCli`      |
| `analyzer` | Powers `--analyze` PRD quality LLM feedback. | `CompletionCli` |

**`[roles.analyzer]` is required in config when `--analyze` is used.** There is no implicit fallback to the executor — analyzer typically wants a cheaper/faster model than the executor, and silently picking one is worse than asking. `zurdo init` writes a sensible default `[roles.analyzer]` block (§11.2) so most users never have to think about it.

`verifier` and `reporter` roles are cut from v1 (see §16).

### 7.4 Executor Prompt Template

Fixed template owned by Zurdo, **not user-configurable in v1**. Sections in fixed order:

```
# Task

<task title>

<task description verbatim>

# Acceptance Criteria

<criteria list, including hint syntax verbatim, with `[manual]` items marked as
"verified by human reviewer">

# Available Skills

<comma-separated list of skill names from **Skills** metadata, if any. The skills
themselves are installed in the provider's native discovery path — see §8.>

# Previous Iteration Feedback

<populated on iteration 2+>

The previous iteration produced changes but criteria did not all pass. Failed checks:

- [shell: cargo test --test auth] → exit code 1
  stdout: <truncated>
  stderr: <truncated>
- [file-exists: src/auth.rs] → file not found at expected path
- [http: GET /health -> 200] → got 503; body: <truncated>

## Prior iteration agent narrative (tail)

<tail of the immediately-prior iteration's .out file, last 4KB, head-truncated.
Sourced from `.zurdo/<slug>/iterations/<task-id>-<attempt-1>.out`.>

Address these failures while keeping previously-passing checks satisfied.

# Instructions

Make the changes needed to satisfy all acceptance criteria. When you believe you are
done, exit cleanly.
```

**Failure stdout/stderr in the prompt** is truncated to 4KB each (same limit as `prd.json` storage). Truncation preserves the tail (most recent output).

### 7.5 Effort → Model Resolution

The `executor` role's provider is fixed by config. The `**Effort**` field selects a **model within that provider** via a per-provider effort_map. Effort never changes the provider, never changes the role.

```toml
[roles.executor]
provider = "anthropic"

[effort_map.anthropic]
low    = "claude-haiku-4-5"
medium = "claude-sonnet-4-7"
high   = "claude-opus-4-7"

[effort_map.codex]
low    = "gpt-5-codex-mini"
medium = "gpt-5-codex"
high   = "gpt-5-codex-pro"
```

The active executor's effort_map is looked up by `[roles.executor].provider`. `**Effort**` is required by the grammar (§2.3.3) and must be a key in the active provider's effort_map (§2.3.9); validity is checked per the active executor at pre-flight, not across all providers. The `executor` role config does not carry a `model` field — the model is always determined by the provider's effort_map.

Each provider that may serve as executor must declare its own `[effort_map.<provider>]` block. The keyset must be consistent across providers if PRDs are to remain portable when the executor provider is swapped — `zurdo init` writes both blocks with matching `low`/`medium`/`high` keys by default.

The `analyzer` role does not consult `effort_map`; it uses its own `model` field directly (§11.2).

### 7.6 Default CLI Invocation

Each adapter has a fixed default invocation; per-provider `extra_args` from config are appended verbatim. The prompt is written to a temp file and piped on stdin in both cases.

**Anthropic (`claude`)** — derived from ralph-loop's proven pattern, extended to request structured output for token-count extraction:

```
claude --dangerously-skip-permissions --print --output-format json --model <resolved-model>
```

**OpenAI (`codex`)** — non-interactive execution with confirmations skipped, structured output requested:

```
codex exec --skip-confirmations --output-format json --model <resolved-model>
```

If either CLI's exact flag spelling differs at implementation time, the adapter MUST match the equivalent non-interactive batch invocation **with structured/JSON output enabled**; the spec records intent (non-interactive, model pinned, no confirmation prompts, structured output so token counts can be parsed per §7.7), not the precise flags. Falling back to plain-text output is permitted but disables token extraction (the adapter emits a `warn`-level diagnostic and continues; `tokens_in`/`tokens_out` are `None` for that iteration).

### 7.7 Output Parsing

- **`AgentCli`:** output is not parsed for content. Zurdo captures stdout/stderr for logs and reads the exit code. The "real output" of an agent invocation is the modified working tree. The adapter additionally extracts `tokens_in` and `tokens_out` from the provider's structured output (§7.6) using a per-provider hardcoded parser. Extraction failure is non-fatal — both fields are set to `None`, downstream cost computation is skipped, and the run continues.
- **`CompletionCli`:** per-provider hardcoded parser in Zurdo's source extracts both the `text` completion and token counts. No configurable regex, no reliance on CLI flags being right beyond the structured-output flag in §7.6.

### 7.8 Pricing Table for Cost Estimation

Zurdo ships a **hardcoded pricing table** keyed by exact model name, mapping each model to `($/MTok_in, $/MTok_out)`. The table covers every model declared in the default `[effort_map.*]` blocks (§7.5) plus the default `[roles.analyzer].model` (§11.2). Models outside the table produce iterations with `cost_usd_est: null`; token counts are still surfaced. Prices are sourced from each provider's public price list and updated as part of each Zurdo release cycle.

```rust
// shape only; concrete numbers live in Zurdo's source and track each release
const PRICING_USD_PER_MTOK: &[(&str, f64 /* in */, f64 /* out */)] = &[
    ("claude-haiku-4-5",   /* in */, /* out */),
    ("claude-sonnet-4-7",  /* in */, /* out */),
    ("claude-opus-4-7",    /* in */, /* out */),
    ("gpt-5-codex-mini",   /* in */, /* out */),
    ("gpt-5-codex",        /* in */, /* out */),
    ("gpt-5-codex-pro",    /* in */, /* out */),
];
```

Cost is computed per iteration as `(tokens_in / 1_000_000) × in_price + (tokens_out / 1_000_000) × out_price` and rounded to four decimal places before being written to `prd.json`. Aggregates (per-task and per-run) are computed by summing per-iteration `cost_usd_est`, treating `null` entries as zero (a single null does not poison the sum, but the totals line in §13.1.5 flags "partial" when any iteration's cost is `null`).

A user-overridable pricing block (`[pricing.<model>]` in `.zurdo/config.toml`) is intentionally **not** part of v1 — see §16. Users who need custom prices (self-hosted endpoints, discount agreements) wait for the v1.x configurable-pricing addition.

---

## 8. Skills System

### 8.1 Storage

Skills live at `.zurdo/skills/<name>/SKILL.md` in **Anthropic-compatible format**: YAML frontmatter (`name`, `description`, optional `allowed-tools`), markdown body, optional sibling files in the same directory.

Flat layout — no `global/` subdir in v1.

### 8.2 Reference

The `**Skills**: <name>, <name>` task metadata field names skills the executor should have available. A reference to a non-existent skill directory is a **parse-time error** caught by the grammar validator. This is why the PRD-01 validator takes the repo root as input, not just a markdown string.

The skill names listed in `**Skills**` are also rendered into the prompt's `# Available Skills` section (§7.4) so the agent knows which skills to invoke.

### 8.3 Installation Model

Zurdo copies skills from `.zurdo/skills/<name>/` into the provider's native skill discovery path. For Anthropic (`claude`), that path is `.claude/skills/<name>/` relative to repo root. Installation runs at two distinct triggers:

- **Bulk install at `zurdo init` (and `zurdo init --sync`).** Copies every `.zurdo/skills/<name>/` directory present at the time of invocation. This is the primary, explicit install path.
- **Auto-install at run-time pre-flight.** When a PRD references a skill via `**Skills**: <name>` and the destination is missing, Zurdo copies it before invoking the executor. This covers the gap between inits when users author new skills or add new PRD references. Idempotent: an already-installed skill is skipped.

```rust
enum SkillInstallOutcome {
    Installed(Vec<PathBuf>),  // paths where skills were installed (or already present)
    NotSupported,             // provider has no skill system; skills silently ignored
}
```

**Collision handling.** At either trigger, if `.claude/skills/<name>/` already exists and lacks the `.zurdo-managed` sentinel file, abort with an error pointing at the conflict (the destination is owned by something outside Zurdo). Zurdo writes the sentinel inside every directory it installs; subsequent installs to the same destination overwrite freely as long as the sentinel is present.

**Per-provider behavior in v1:**

| Provider             | `install_skills` behavior                                                                                                           |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Anthropic (`claude`) | Copy `.zurdo/skills/<name>/` → `.claude/skills/<name>/` (with `.zurdo-managed` sentinel). Claude discovers natively. → `Installed`. |
| OpenAI (`codex`)     | Copy `.zurdo/skills/<name>/` → `.codex/skills/<name>/` (with `.zurdo-managed` sentinel). Codex discovers natively. → `Installed`.   |

### 8.4 Bundled Opinionated Skills

Zurdo ships a small set of bundled opinionated skills (programming-language helpers, tool-execution patterns) compiled into the binary. `zurdo skills list` shows what's available; `zurdo skills install <name>` copies a bundled skill into `.zurdo/skills/<name>/` **and** immediately installs it into the provider's discovery path (so PRDs can reference it right away). Bundled skills are seed templates — once installed, they live under `.zurdo/skills/` like user-authored skills and may be edited freely.

### 8.5 Unsupported Providers

When `install_skills` returns `NotSupported`, Zurdo logs a `warn`-level message per skill ("skill `auth` ignored: provider `xyz` does not support skills") and proceeds. The task may fail for lack of the skill's guidance, but that surfaces as a normal task failure.

**No inlining fallback in v1.** Inlining `SKILL.md` contents into the prompt for unsupported providers was cut: it complicates the prompt template, doesn't carry sibling files, and creates two divergent skill-authoring conventions. Skill-using PRDs implicitly require a skill-capable provider.

---

## 9. PRD Analysis (`--analyze`)

`--analyze` runs a comprehensive pre-flight check on a PRD and exits without executing any tasks. It is the dedicated "is this PRD well-formed and runnable?" pass, surfacing structural problems and quality issues in one place before a user commits compute to a real run. With `--analyze --fix`, the same pass becomes the quality signal for an autonomous LLM-driven refinement loop that proposes a cleaned-up PRD as a sibling file (§9.7).

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
| Effort declarations         | `**Effort**` is one of the keys in `[effort_map.<active-executor-provider>]`                  | Deterministic |
| Skills references           | Every `**Skills**: <name>` resolves to an existing `.zurdo/skills/<name>/` dir                | Deterministic |
| Acceptance criteria quality | Criteria are concrete and testable (not vague like "works well")                              | LLM           |
| Task scope                  | Tasks aren't oversized, ambiguous, or doing too many things                                   | LLM           |

The `analyzer` role is always invoked if configured. There is no flag to skip the LLM portion — if you don't want LLM passes, don't configure an `analyzer` role.

### 9.3 Severity Tiers

- **error** — the PRD will not run correctly as-is (cycle, dangling dep, missing required field, malformed criterion hint, dangling skill ref).
- **warning** — the PRD will run but something is likely wrong (vague criterion, oversized task).
- **info** — advisory observation (resolved execution order, etc.).

### 9.4 Output

Output goes to stdout: a leading stats banner, per-task findings grouped by task with severity tags and per-finding suggestions, and a color-coded verdict block at the end. Global findings appear in a leading section under the stats. Color follows §13.3 (TTY-detect, honors `NO_COLOR` and `--no-color`).

```
== Stats ==
  Total tasks:        7
  Effort:             low=1, medium=4, high=2
  Avg criteria/task:  4.6
  Skills referenced:  auth, rate-limit-policy
  Categories:
    Backend:       4
    Frontend:      2
    Uncategorized: 1

== Global ==
[info] Execution order: task-1 → task-2 → task-4 → task-3
[error] Cycle detected: task-5 → task-6 → task-5
  suggestion: break the cycle by removing one direction of the dependency.

== task-1: Set up auth ==
[error] Missing required field: Effort
  suggestion: add `**Effort**: <key>` in the metadata block (allowed keys come from [effort_map.<executor>]).
[warning] Criterion is vague and not directly testable: "should handle edge cases gracefully"
  suggestion: rewrite as a concrete check, e.g. `[shell: cargo test edge_cases]` or `[file-exists: ...]`.

== task-2: Add login endpoint ==
[warning] Criterion is vague and not directly testable: "login should feel snappy"
  suggestion: define a measurable threshold, e.g. `[http: GET /login -> 200]` plus a `[shell:]` timing check.
[info] 3 criteria, all with type hints

═══ Verdict ═══
✗ NOT READY — 2 errors, 2 warnings, 1 info
  Fix the errors above, then re-run --analyze.
```

Severity tags are color-coded when color is enabled: `[error]` red, `[warning]` yellow, `[info]` cyan. `suggestion:` lines are dimmed.

The **Stats** banner is always emitted, even when no `Category` field is used (the Categories block is omitted in that case). When at least one task declares `**Category**`, tasks without it group under `Uncategorized`. The banner is deterministic and computed before any LLM calls run, so it appears immediately on `--analyze` invocation.

The verdict block is always emitted at the end, in one of three shapes:

| Verdict                  | Color  | Trigger                  | Recommendation line                                  |
| ------------------------ | ------ | ------------------------ | ---------------------------------------------------- |
| `✓ READY TO RUN`         | green  | no errors, no warnings   | _(no recommendation line)_                           |
| `⚠ READY WITH WARNINGS` | yellow | no errors, ≥1 warning    | `Address warnings before running for best results.`  |
| `✗ NOT READY`           | red    | ≥1 error                 | `Fix the errors above, then re-run --analyze.`       |

Counts in the verdict line aggregate global + per-task findings.

### 9.4.1 Per-Finding Suggestions

Every `error` and `warning` finding carries a `suggestion:` line beneath it. `info` findings do not. Suggestion text is sourced from one of two places:

1. **Static lookup (always available).** A fixed table keyed off finding type, defined in Zurdo's source. Deterministic, no network call. Static suggestions are short (one line) and prescriptive ("add field X", "rewrite as Y", "break the cycle by …").
2. **LLM elaboration (only when `[roles.analyzer]` is configured and the finding is a `warning`).** For each warning whose static suggestion is generic, Zurdo asks the analyzer for a finding-specific elaboration sourced from the actual criterion/task text. The elaboration replaces the static text when received; on analyzer failure or timeout, the static fallback is shown. One round-trip per eligible warning. Errors always use the static suggestion (cheaper, less ambiguous, faster verdict).

### 9.5 Exit Codes

When `--fix` is **not** set, exit codes are binary:

- `0` — no errors (warnings and info findings do not fail).
- non-zero — one or more errors present.

When `--fix` is set, the loop introduces additional terminal codes (`6` cap exhausted, `7` thrash, `8` parse failure). The full table lives in §12.3 and is enumerated alongside loop semantics in §9.7.4.

The verdict color always reflects severity in three tiers regardless of mode. `--analyze` without `--fix` is safe to drop into CI as a PRD lint step.

### 9.6 Behavior Notes

- Analyze does not require network access for deterministic checks.
- LLM checks (criteria-quality, task-scope) and per-warning LLM elaboration shell out to the `analyzer` provider the same way executor calls do.
- If `[roles.analyzer]` is not present in config, `--analyze` fails fast before any checks run with "configure `[roles.analyzer]` to enable LLM analysis." If it is configured but the binary is missing from `PATH`, fail with the same "missing binary" message the executor pre-flight emits.
- The verdict block always renders, regardless of analyzer presence. Without the analyzer, every suggestion is the static fallback.
- Skills are not installed during analyze — the analyzer operates on the raw PRD only.

### 9.7 Iterative Refinement (`--fix`)

`--analyze --fix` enables an autonomous loop that iterates an LLM against the analyze pass until warnings converge or a termination condition trips. The loop reuses the `[roles.analyzer]` role (via `CompletionCli`) — no `AgentCli` plumbing, no working-tree mutation. The fixer rewrites the PRD text in its response; Zurdo writes the result to a sibling `<prd>.proposed.md` next to the original PRD, never overwriting the original without explicit user confirmation.

#### 9.7.1 Invocation

```
zurdo --analyze --fix --max-iterations N <prd>
```

- `--fix` requires `[roles.analyzer]` (same dependency as the existing LLM portion of `--analyze`, §9.2). If missing, fail fast with the standard "configure `[roles.analyzer]`" message and exit `3`.
- `--max-iterations N` is shared with `zurdo run` semantically (a budget cap) but only takes effect for the fix loop when `--fix` is set. **Default in fix mode: `5`** (matches `[defaults] max_attempts`, §11.2). `--max-iterations 0` is a flag error in fix mode (`0` means "unlimited" only for `run`; fix always wants a finite cap).
- `--fix` without `--analyze` is a flag error.
- `--no-prompt` suppresses the final approval prompt (§9.7.4); the proposed file is the artifact.

#### 9.7.2 Pre-flight

Before the loop starts, Zurdo runs the §9.2 analyze pass once and branches on the result:

- **Errors present (`errors > 0`).** Refuse to start. Print the errors with their static suggestions and exit `2`. Message: "`--fix` addresses warnings only — fix errors first." Errors are mechanical (cycles, missing required fields, dangling deps) and one-line edits; the loop does not attempt them.
- **No findings (`errors == 0 && warnings == 0`).** Print "already clean" and exit `0`. No proposed file is written.
- **Warnings present, no errors.** Enter the loop at iteration 1.

#### 9.7.3 Per-Iteration Lifecycle

For each iteration `n` (starting at 1):

1. **Build the fixer prompt** containing:
   - Full PRD verbatim at iteration `n-1` (or the original PRD on iteration 1).
   - Full analyze output for iteration `n-1`: every warning's text, hint-or-task reference, and static + LLM-elaborated suggestion (§9.4.1).
   - A **preservation block**: task IDs, declaration order, and the literal values of `**Effort**` and `**Depends-on**` MUST NOT change. The LLM may edit criterion text, descriptions, and add/remove individual criteria; it may not rename tasks, reorder them, or restructure the dep graph. Violations are caught at parse time (§9.7.4 parse-failure path).
2. **Invoke `[roles.analyzer]` via `CompletionCli`.** Record `tokens_in` / `tokens_out` / `cost_usd_est` for this iteration.
3. **Parse the LLM's response** as a PRD under the strict grammar (§2.2). If parsing fails, halt per §9.7.4 (parse-failure exit).
4. **Persist iteration artifacts** to `.zurdo/<slug>/analyze-iterations/<n>.md` (full PRD content) and `.zurdo/<slug>/analyze-iterations/<n>.findings.json` (the analyze output that drove this iteration — i.e., iteration `n-1`'s findings).
5. **Re-run the analyze pass** against iteration `n`'s PRD to produce its warning set.
6. **Render the per-iteration progress block** to stdout (§9.7.5).
7. **Evaluate termination conditions** (§9.7.4). Halt on the first match; otherwise loop.

#### 9.7.4 Termination Conditions

The loop halts on the first of the following four conditions:

| Condition                                                 | Exit | Outcome                                                                                                                                                                  |
| --------------------------------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Warning count reaches `0`                                 | `0`  | Converged clean. Iteration `n`'s PRD becomes `<prd>.proposed.md`.                                                                                                        |
| Iteration count reaches `N` with warnings remaining       | `6`  | Cap exhausted. Iteration `n`'s PRD becomes `<prd>.proposed.md`. Stderr hint: "raise the cap and re-run — `--max-iterations <N+5>`."                                       |
| Warning count non-decreasing across the last 3 iterations | `7`  | Thrash. Iteration `n`'s PRD becomes `<prd>.proposed.md`. Stderr emits the §9.7.6 diagnostic.                                                                             |
| LLM produced an unparseable PRD                           | `8`  | Halted. Iteration `n-1`'s PRD becomes `<prd>.proposed.md` (last good). The unparseable response is preserved under `.zurdo/<slug>/analyze-iterations/<n>.parse-error.txt`. |

**Thrash window** is 3: thrash fires at iteration `n` iff `warnings[n] >= warnings[n-1] >= warnings[n-2]`. The window is not configurable in v1. The first 2 iterations cannot trigger thrash (insufficient history).

**Parse failure on iteration 1** (no prior good iteration exists): no proposed file is written; the original PRD is unchanged; exit `8` still fires. No retry with the parse error fed back — v1 chose predictability over recovery thrash.

**After any non-error halt:**

1. Write `.zurdo/<slug>/analyze-iterations/summary.json` with total iterations, exit code, exit condition, per-iteration warning counts, per-warning trajectory (§9.7.6), aggregate `tokens_in` / `tokens_out` / `cost_usd_est`.
2. Write the promoted iteration's full PRD to `<prd>.proposed.md`, silently overwriting any prior `.proposed.md`. The sibling file lives next to the original PRD, not inside `.zurdo/<slug>/`.
3. On TTY without `--no-prompt`, render a unified diff `<prd>.md` → `<prd>.proposed.md` and prompt `Apply these changes? [y/N]`. `y` performs `mv <prd>.proposed.md <prd>.md`; default (or anything not `y`/`Y`) leaves the proposed file in place. The prompt does not change the exit code.
4. On non-TTY or `--no-prompt`, no prompt renders; the proposed file is the artifact.

A subsequent `--analyze` against the original PRD still reports the unchanged findings (the user has not accepted). A subsequent `--analyze` against the proposed file reports the new findings. A subsequent `--analyze --fix` overwrites the prior `<prd>.proposed.md` and the prior `analyze-iterations/` contents.

#### 9.7.5 Run-Progress Output

While the loop runs, stdout renders a per-iteration block that tracks individual warnings using a status tag derived from the prior iteration's warning set. Warnings are identified by `(task_id, finding_category, criterion_index)` so the LLM's rewording of a flagged criterion is recognized as a continuation of the same warning rather than a new finding.

```
─── analyze --fix: prds/auth.md ──────────────────────────────────
  → iteration 2 of 5
  ✓ analyzer completed: 3.8s — tokens in=2,104 out=3,891 — $0.05 est.
  iter 2: 4 warnings remaining
    [RESOLVED] task-auth — criterion "should handle edge cases gracefully"
    [PERSISTS] task-login — criterion "login should feel snappy"  (2 iters)
    [PERSISTS] task-login — task scope too broad                  (2 iters)
    [RESOLVED] task-rate-limit — criterion "is fast enough"
    [NEW]      task-admin — criterion "looks good"
```

Status tags:

| Tag          | Meaning                                                                                                                                              |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `[NEW]`      | Warning appeared this iteration; not present in the prior iteration's set. Signals a regression introduced by the rewrite.                           |
| `[PERSISTS]` | Warning was present last iteration and is still present. Carries the iteration-count it has been stuck across.                                       |
| `[RESOLVED]` | Warning was present last iteration and is gone this iteration. Logged on the iteration where it disappears, then dropped from subsequent blocks.     |

Glyphs (`→`, `✓`) follow §13.1.3. Color follows §13.3 (TTY auto-detect, honors `NO_COLOR` and `--no-color`). On non-TTY, the same content renders without spinners but with the same status tags. `--no-progress` suppresses the entire run-progress stream; the audit trail under `.zurdo/<slug>/analyze-iterations/` is still written.

The per-iteration `tokens in=… out=… — $… est.` line is omitted entirely when the analyzer adapter could not extract token counts (§7.7); the `— $… est.` suffix is dropped when the analyzer's model is absent from the pricing table (§7.8). Aggregate tokens/cost render in the end-of-loop summary regardless.

#### 9.7.6 Thrash Diagnostic (Exit `7`)

When the loop halts on thrash, stderr emits a diagnostic listing the warnings that persisted through the final 3 iterations, with the criterion or task text the LLM produced at each iteration so the user can see what the fixer kept trying:

```
Fix loop halted: warning count non-decreasing across 3 iterations.
Warnings persisted through final 3 iterations:

  task-login — criterion "login should feel snappy"
    iter 3: "login response is fast"           (still vague)
    iter 4: "login meets performance goals"    (still vague)
    iter 5: "login should be performant"       (still vague)
    hint: add a measurable threshold, e.g. [shell:] timing check or
          [http: GET /login -> 200] with a latency assertion.

  task-login — task scope too broad
    iter 3: (no text change)
    iter 4: (no text change)
    iter 5: (no text change)
    hint: split into smaller tasks with narrower acceptance criteria.

Proposed file written to prds/auth.proposed.md (resolved warnings preserved).
```

The `hint:` lines are sourced from the same static-suggestion table used in §9.4.1. Per-warning trajectory is recorded in `summary.json` (§9.7.7) on every non-error halt, not only exit `7`, so external tooling can render the same view.

#### 9.7.7 Audit Trail Layout

```
.zurdo/<slug>/analyze-iterations/
├── 1.md                   # full PRD content at iteration 1
├── 1.findings.json        # analyze output for the *original* PRD (drove iter 1's prompt)
├── 2.md
├── 2.findings.json        # analyze output for iter 1 (drove iter 2's prompt)
├── ...
├── <N>.parse-error.txt    # only on exit `8`; the unparseable response + parse error
└── summary.json           # roll-up: iterations, exit code, exit condition, per-warning
                           # trajectory, aggregate tokens & cost
```

The audit trail is overwritten on each `--fix` invocation against the same PRD (no archival of prior fix runs in v1). The contents are unrelated to `progress.log` and are never read by `zurdo run`. Users may delete `.zurdo/<slug>/analyze-iterations/` at any time; doing so does not affect the original PRD, the proposed PRD, or any in-flight `prd.json`.

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
- Per-task and per-run **token usage and cost estimate** (sourced from `tokens_in`/`tokens_out`/`cost_usd_est` in `prd.json`). The per-run totals carry a `partial: true` flag when any iteration had `cost_usd_est: null`.
- Criteria hotspots (criteria that failed most across attempts).
- Provider/model usage summary across the run.
- Stalled task detection (tasks that hit max attempts without progress).
- Per-task iteration narrative rendered deterministically from `iterations` in `prd.json` (e.g., "Iteration 1: 2/5 criteria passed; failing: cargo test, file-exists src/auth.rs. Iteration 2: 5/5 passed."). No LLM round-trip.
- Distinction between `failed` and `blocked-by-dependency` so root causes are clear.
- **Category grouping (optional).** When at least one task declares `**Category**`, the report renders an additional "By category" section that groups task rows under their declared category (uncategorized tasks under `Uncategorized`). Categories never replace the canonical per-task list — they are a supplementary view. When no task declares `**Category**`, the section is omitted entirely.

---

## 11. Configuration

### 11.1 Scope

**Per-repo only in v1.** `.zurdo/config.toml` at the repo root. No user-level config, no per-PRD overrides. If shared defaults across repos are needed, the user manages that out-of-band.

**Config is required.** A missing `.zurdo/config.toml` is an error. Zurdo provides `zurdo init` to write a sensible default with comments — the same file works as documentation.

### 11.2 Config File

```toml
# .zurdo/config.toml

[roles.executor]
provider = "anthropic"          # or "codex"
# model is determined by [effort_map.<provider>]; not configured here.

[roles.analyzer]
provider = "anthropic"          # required when --analyze is used
model    = "claude-haiku-4-5"   # used directly (effort_map does not apply to analyzer)

[effort_map.anthropic]
low    = "claude-haiku-4-5"
medium = "claude-sonnet-4-7"
high   = "claude-opus-4-7"

[effort_map.codex]
low    = "gpt-5-codex-mini"
medium = "gpt-5-codex"
high   = "gpt-5-codex-pro"

[defaults]
max_attempts         = 5        # applies when a task omits **Max-Attempts**
max_total_iterations = 0        # global run-wide budget (0 = unlimited; see §5.6).
                                # CLI --max-iterations overrides this.

[timeouts]
criterion_seconds = 300         # per-check, shell + http only
agent_seconds     = 1800        # applies when a task omits **Agent-timeout**

[providers.anthropic]
cli        = "claude"           # binary on PATH
extra_args = []                 # appended after Zurdo's default args (§7.6)

[providers.codex]
cli        = "codex"            # binary on PATH
extra_args = []                 # appended after Zurdo's default args (§7.6)
```

`zurdo init` writes both `[providers.*]` blocks and both `[effort_map.*]` blocks by default, even if only one provider is referenced by roles. Switching the executor between providers is then a one-line `provider = "..."` change with no further plumbing.

### 11.3 Pre-flight Validation

At run start, Zurdo validates:

1. `.zurdo/config.toml` exists and parses.
2. Every role's `provider` exists in the `[providers.*]` section.
3. Every provider's `cli` binary is present on `PATH`. Missing binary → fail fast with a hint pointing to the role config and the install path for the provider.
4. The PRD parses cleanly under the strict grammar (including resolution of every `**Skills**` reference to a real `.zurdo/skills/<name>/` directory).
5. The `.zurdo/<slug>/lock` file is either absent or stale (§5.1).
6. If `.zurdo/<slug>/prd.json` exists, run the interactive resume prompt (§6.2 step 0) unless bypassed by `--resume`, `--reset`, `--no-prompt`, or a non-TTY stdin. The user's choice may short-circuit the remaining pre-flight (e.g., `[A] Abort` exits immediately; `[X] Reset` proceeds to step 7 with archived state).
7. The PRD's `prd_hash` matches the recorded hash in `prd.json` (if present) — else abort with diff and require `--reset` (or the `[X]` choice from step 6).
8. For every `**Skills**` reference, the destination path is either already present (passes) or installable without colliding with a non-Zurdo-managed skill (§8.3). Missing destinations trigger auto-install at this point.
9. If `--analyze` is set, `[roles.analyzer]` is present in config.

Credentials (env vars like `ANTHROPIC_API_KEY`) are **not** validated. If the binary is present but auth is broken, that surfaces on the first agent invocation as a normal failed iteration with the CLI's own stderr captured.

---

## 12. CLI Surface

### 12.1 Subcommands

| Subcommand                    | Purpose                                                                  |
| ----------------------------- | ------------------------------------------------------------------------ |
| `zurdo init`                  | Write a default `.zurdo/config.toml` with comments and install every `.zurdo/skills/<name>/` into the provider's discovery path. |
| `zurdo init --sync`           | Re-install every `.zurdo/skills/<name>/` into the provider's discovery path without rewriting `config.toml`. |
| `zurdo run <prd>`             | Run the PRD (default if no subcommand and a positional PRD is given).    |
| `zurdo report <prd>`          | Generate a report from `prd.json`.                                       |
| `zurdo validate <prd>`        | Run deterministic grammar/structural checks only (no LLM, no execution). |
| `zurdo skills list`           | List bundled opinionated skills available for install.                   |
| `zurdo skills install <name>` | Copy a bundled skill into `.zurdo/skills/<name>/` and immediately into the provider's discovery path. |

### 12.2 Flags

| Flag                        | Effect                                                                                       |
| --------------------------- | -------------------------------------------------------------------------------------------- |
| `--analyze`                 | Run full pre-flight (deterministic + LLM) and exit. See §9.                                  |
| `--fix`                     | With `--analyze`: run the iterative refinement loop (§9.7) instead of exiting after one pass. Writes the proposed PRD to `<prd>.proposed.md`. Requires `[roles.analyzer]`. Flag error without `--analyze`. |
| `--reset`                   | Archive old state and start over (on `prd_hash`/`schema_version` mismatch, or unconditionally). See §5.5. Skips the interactive resume prompt. |
| `--resume`                  | Skip the interactive resume prompt (§6.2 step 0) and resume from existing state. No-op when no state exists. |
| `--no-prompt`               | Suppress all interactive prompts (CI-safe). Resume prompt defaults to Resume; other prompts behave as the documented bypass. |
| `--max-iterations N`        | Cap total agent invocations across the run. Overrides `[defaults] max_total_iterations`. `0` = unlimited. See §5.6. |
| `--format <json\|md>`       | Output format for `zurdo report`. Default `json`.                                            |
| `--log-file <path>`         | Tee diagnostic log to a file in addition to stderr. (Independent of `progress.log`.)         |
| `-v` / `-q`                 | Aliases for `--log-level=debug` / `--log-level=warn`. Mutually exclusive with `--log-level`. |
| `--log-level=<level>`       | One of `error`, `warn`, `info`, `debug`, `trace`. Default `info`.                            |
| `--log-format=<text\|json>` | Diagnostic log format. Default `text`. Does not affect the run-progress stream.              |
| `--no-progress`             | Suppress the stdout run-progress stream entirely (banner, headers, summary). See §13.1.      |
| `--quiet-agent`             | On TTY, suppress the live tee of agent stdout/stderr; spinner + heartbeats remain. See §13.2.|
| `--no-color`                | Disable ANSI color codes in all stdout streams (`run`, `--analyze`, `report`). See §13.3.    |

### 12.3 Exit Codes

| Code | Meaning                                                          |
| ---- | ---------------------------------------------------------------- |
| `0`  | Success.                                                         |
| `1`  | General failure (any unhandled error).                           |
| `2`  | PRD parse / validation error (analyze findings, grammar errors). |
| `3`  | Pre-flight failure (missing config, missing binary, lock held).  |
| `4`  | State mismatch requiring `--reset`.                              |
| `5`  | One or more tasks finished `failed` or `blocked-by-dependency`.  |
| `6`  | Iteration budget exhausted. From `zurdo run --max-iterations` (§5.6) or `zurdo --analyze --fix --max-iterations` with warnings remaining (§9.7.4). |
| `7`  | `--analyze --fix` thrash detected — warning count non-decreasing across the last 3 iterations (§9.7.4). |
| `8`  | `--analyze --fix` halted — LLM produced an unparseable PRD; last-good iteration preserved (§9.7.4). |

---

## 13. Logging

Zurdo emits **three** output streams. They serve different audiences and have different lifecycles. They are deliberately independent: a user can silence one without affecting the others.

| Stream             | Sink                            | Audience              | Purpose                                                              |
| ------------------ | ------------------------------- | --------------------- | -------------------------------------------------------------------- |
| **Run progress**   | stdout                          | Human at terminal     | Live, color-coded, ralph-loop-style narration of the running PRD.    |
| **Diagnostic log** | stderr (and optional file tee)  | Human reader / ops    | Real-time diagnostics. Configurable level/format.                    |
| **Event log**      | `.zurdo/<slug>/progress.log`    | Resume + tooling      | Structured JSONL event stream for crash recovery and external tools. |

### 13.1 Run Progress Output (stdout)

The run progress stream is the primary surface for a human watching `zurdo run` in their terminal. It is designed to be **ralph-loop-informative**: a startup banner, per-task and per-iteration headers, per-criterion live pass/fail, the agent's own output streamed live, and a final summary table. Always on stdout; never interleaved with the diagnostic log.

#### 13.1.1 Startup Banner

Emitted once at the start of `zurdo run`:

```
═══════════════════════════════════════════════════════════
  Zurdo v1.0.0
  PRD:      prds/auth.md
  Slug:     auth-a1b2
  Executor: anthropic (effort_map: low=claude-haiku-4-5,
                                   medium=claude-sonnet-4-7,
                                   high=claude-opus-4-7)
  Analyzer: anthropic (claude-haiku-4-5)
  Tasks:    7 (4 pending, 2 passed, 1 blocked-by-dependency)
  Resumed:  yes — picking up from last successful task-2
═══════════════════════════════════════════════════════════
```

The slug, model map, and prior-state counts reflect what Zurdo resolved at pre-flight. The `Resumed:` line only appears when resuming an existing `.zurdo/<slug>/prd.json`.

#### 13.1.2 Per-Task Section

Each task opens with a divider and a header line:

```
─── task-auth: Set up authentication ─── effort=medium, deps=[]
```

Status transitions emit indented status lines under the header. Pre-flight short-circuit:

```
  → running pre-flight criteria
  ✓ pre-flight passed: 0 iterations needed
```

Otherwise the iteration loop opens:

```
  → iteration 1 of 5 (max-attempts=5, agent-timeout=30m)
```

#### 13.1.3 Per-Iteration Block

While an iteration runs (TTY only), Zurdo refreshes a status line in place with a spinner, elapsed time, and a running byte count of the agent's stdout:

```
  ⠋ task-auth iter 1 — claude-sonnet-4-7 — 12s — 4.1 KB stdout
```

When the agent finishes, the spinner line is committed and Zurdo emits the agent close-out and per-criterion results:

```
  ✓ agent completed: exit=0, 47s, 18.4 KB stdout, 0.2 KB stderr
                     tokens in=3,421 out=8,209 — $0.12 est.
  → running 5 criteria
    ✓ shell: cargo build (1.2s)
    ✓ shell: cargo test (8.4s)
    ✗ shell: cargo clippy -- -D warnings (3.1s)
      stderr tail: error: this loop never actually loops
    ✓ file-exists: src/auth.rs
    ⊘ manual: rate limiting confirmed under load
  iteration 1: 3/5 criteria passed; will retry
```

Glyphs and colors:

| Glyph | Color  | Meaning                                                              |
| ----- | ------ | -------------------------------------------------------------------- |
| `→`   | cyan   | Action started.                                                      |
| `⠋…⠿` | cyan   | Spinner; TTY only. On non-TTY no spinner is emitted.                 |
| `✓`   | green  | Pass / success.                                                      |
| `✗`   | red    | Fail.                                                                |
| `⊘`   | blue   | Skipped / manual / not-run.                                          |
| `⚠`   | yellow | Warning (transient retry, timeout, stale-lock takeover).             |

Failing-criterion lines include the tail (last ~200 chars) of the criterion's stderr (or stdout if stderr is empty), one line deep, dimmed. The full untruncated output remains in `prd.json` per §5.2 and the iteration files per §5.3.

The `tokens in=… out=… — $… est.` continuation line is omitted entirely when the adapter could not extract token counts (per §7.7). When tokens are present but the model is absent from the pricing table (§7.8), the `— $… est.` suffix is dropped and only `tokens in=… out=…` is rendered.

#### 13.1.4 Task Close

A single committed line per task, colored by terminal status:

```
  ✓ task-auth: passed in 2 iterations (8m 14s)
```

- `passed` / `passed-pending-review` → green
- `failed` → red
- `blocked-by-dependency` → gray (`⊘` glyph)

#### 13.1.5 Final Summary Table

At the end of the run (also emitted on Ctrl-C-first-press graceful exit, reflecting partial state):

```
═══ Run Summary ═══
  task-id              status                  attempts   wall-clock   criteria
  ─────────────────────────────────────────────────────────────────────────────
  task-auth            passed                  2/5        8m 14s       5/5
  task-login           passed-pending-review   1/5        3m 02s       4/4
  task-rate-limit      failed                  5/5        12m 41s      3/4
  task-admin-ui        blocked-by-dependency   0/5        —            —
  ─────────────────────────────────────────────────────────────────────────────
  totals               1 passed, 1 pending-review, 1 failed, 1 blocked     24m 57s
  iterations           8 used (of unlimited)
  tokens & cost        in=142,308 out=384,902 — $0.34 est.
```

Columns are fixed: `task-id | status | attempts (used/max) | wall-clock | criteria (passed/total)`. The status column is the only one that carries color. Values are sourced from `prd.json` at the moment of emission. Tasks that never ran (e.g., `blocked-by-dependency` or budget-truncated `pending`) render `—` in `wall-clock` and `criteria`.

Two trailing aggregate lines follow the totals row:

- **`iterations`** — total real attempts consumed across the run, expressed as `<used> used (of <cap>)` or `<used> used (of unlimited)` when no `--max-iterations`/`max_total_iterations` cap is set (§5.6).
- **`tokens & cost`** — aggregate tokens in / out and summed `cost_usd_est` (§7.8). When any iteration had `cost_usd_est: null`, the line is suffixed `($X.XX est., partial — N iterations had no pricing)`. When *every* iteration was pricing-less, the cost portion is dropped and only tokens render. When no iteration had token data either, the line is omitted entirely.

#### 13.1.6 Sticky Dashboard Header (TTY only)

On terminals that support ANSI cursor save/restore (`\033[s` / `\033[u`), Zurdo pins a 2-line summary at the top of the scroll buffer and refreshes it at iteration boundaries:

```
─── task progress: 3/7 passed, 1 failed, 1 in-progress, 2 pending ───────────
─── iter 8/15 — 12m 43s — tokens 142,308 — $0.34 est. ────────────────────────
```

- **Line 1** — live aggregate task-status counts. Color matches the per-status legend (§13.1.3): green/yellow/red/gray.
- **Line 2** — iteration counter (`iter <used>/<cap>` or `iter <used>/∞` when no `--max-iterations` cap is in effect), wall-clock since run start, cumulative tokens, cumulative cost. The cost token is omitted when no iteration so far has had pricing.
- **Repaint cadence.** The header repaints on every `task_status` event and at run start/end. It does **not** repaint on each line of teed agent output — that would fight with the live agent tee (§13.2).
- **Non-TTY.** The header is omitted entirely; the run-progress stream falls back to its line-scrolling default.
- **Capability detection.** Zurdo probes terminfo (`tput cup` / `tput sc`) at startup. Terminals without cursor-save/restore support get no header (no broken half-render).
- **Suppression.** `--no-progress` removes the header along with the rest of the run-progress stream. `--no-color` strips color codes but keeps the header structure.

#### 13.1.7 Next-Step Suggestion

When a run ends non-zero, the summary block emits a single tailored suggestion line below the totals (cyan when color is enabled):

| Exit cause                                                          | Suggestion line                                                                       |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Exit `5` — one or more tasks `failed`                               | `Next: fix root causes and re-run — zurdo run <prd>`                                  |
| Exit `5` — only `blocked-by-dependency` (no genuine `failed` tasks) | _(suggestion omitted — the root-cause `failed` already carries one)_                  |
| Exit `6` — global iteration budget exhausted                        | `Next: raise the cap and re-run — zurdo run <prd> --max-iterations <used + 10>`       |
| Exits `2`, `3`, `4` — parse / pre-flight / state-mismatch failures  | _(suggestion omitted — the underlying error message already advises)_                 |
| Exit `0` — success                                                  | _(suggestion omitted)_                                                                |

The suggestion text is fixed (no LLM call) and contains a copy-pasteable command derived from the actual PRD path passed on the CLI and, for exit `6`, the actual iteration count consumed. When the run was launched without a positional PRD argument (e.g., via shell alias), `<prd>` is rendered as `<prd-path>` and the user is expected to fill it in.

### 13.2 Agent Output Streaming

When an agent is running, Zurdo can also tee the agent's own stdout/stderr to the user's terminal — the ralph-loop "watch the agent think" experience. Behavior is TTY-dependent:

- **TTY, default (live tee).** Agent stdout/stderr is teed in real time, **dimmed (gray) and indented 4 spaces** under the iteration spinner line. The spinner line stays pinned above the tee. The full untruncated copy still lands in `.zurdo/<slug>/iterations/<task-id>-<attempt>.{out,err}` per §5.3.
- **Non-TTY (piped).** No live tee — interleaving would corrupt downstream consumers. A heartbeat line is emitted every 10s instead: `task-auth iter 1 — claude-sonnet-4-7 — 47s — 18.4 KB stdout`. Full output still goes to the iteration file.
- **`--quiet-agent` (TTY opt-out).** Suppresses the live tee on TTY; the spinner status line and heartbeats remain. Iteration files unaffected. Useful for chatty agents or long-running batches.

The tee is best-effort: Zurdo strips cursor-control ANSI sequences from the agent's output before tee'ing (color codes are preserved if the agent emits them, but cursor moves, screen clears, and alternate-screen toggles are dropped) to keep its own status line intact.

### 13.3 Color and TTY Detection

A single color decision applies to every stdout stream Zurdo emits (`run`, `--analyze`, `report`):

- Color is **enabled by default when stdout is a TTY**.
- Color is **disabled by default when stdout is not a TTY** (piped, redirected, captured).
- The `NO_COLOR` environment variable, when set to any non-empty value, disables color unconditionally (per <https://no-color.org>).
- The `--no-color` flag disables color unconditionally; takes precedence over auto-detect.
- Spinners and in-place line refreshes are **TTY-only**. On non-TTY, status lines simply scroll with no cursor control. `--no-color` does not disable spinners (it only strips ANSI color codes); to silence the live spinner without losing the stream, redirect stdout.
- To suppress the run-progress stream entirely (keeping diagnostic log and `progress.log`), pass `--no-progress` or redirect stdout (`> /dev/null`).

### 13.4 Diagnostic Log (stderr)

Independent of the run-progress stream. The diagnostic log is the place for structured machine-readable events, warnings about Zurdo's own internals, and verbose debugging output. It never duplicates content from the run-progress stream — the two streams are complementary, not redundant.

Controls:

- **Default sink:** stderr.
- **`--log-file <path>`:** opts in to a file tee (does not redirect; stderr still receives the stream).
- **`-v` / `-q`:** shorthand for `--log-level=debug` / `--log-level=warn`. Passing both `-v`/`-q` and `--log-level` is an error rather than a precedence rule.
- **`--log-format=<text|json>`:** orthogonal to level. Only affects the diagnostic log; the run-progress stream is always human-formatted.

Standard ladder:

| Level   | What goes there                                                                          |
| ------- | ---------------------------------------------------------------------------------------- |
| `error` | Fatal conditions (missing binary, unparseable config, lock held by live pid).            |
| `warn`  | Soft failures (transient retry, skill provider mismatch, criterion timeout, stale lock). |
| `info`  | Task lifecycle (default).                                                                |
| `debug` | Iteration internals (per-iteration timings, criteria durations).                         |
| `trace` | Everything, including full prompts and untruncated criterion output.                     |

### 13.5 Event Log (`.zurdo/<slug>/progress.log`)

`progress.log` event semantics are defined in §6.1. The event log is unrelated to the diagnostic log level and to the run-progress stream — every event is always written, regardless of `--log-level`, `--no-progress`, or `--quiet-agent`.

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
- `progress.log` event encoding/decoding
- Resume reconciliation algorithm

### 14.2 Layer 2: Integration Tests with `MockAgentCli`

Scripted invocation sequences via a `MockAgentCli`. Subprocesses and filesystem are all real (tmpdir-backed repos); only the agent is faked. Covers:

- Runner loop end-to-end
- Crash recovery (no git involvement)
- Ctrl-C handling (first and second press)
- Iteration limits
- Pre-flight criteria pass (iteration 0)
- All-manual task short-circuit
- `prd_hash` mismatch and `--reset`
- Transient-error classification and retry/backoff
- Lock acquisition + stale-lock takeover
- Skill installation (copy into `.claude/skills/`, collision detection, sentinel write)

`MockAgentCli` and shared test infrastructure live in PRD-06 alongside the `AgentCli` trait. PRDs 04, 05, 07, 08 depend on PRD-06 for their integration tests, so PRD-06 must ship its mock infrastructure early (**mock first, real Anthropic adapter second**).

### 14.3 Layer 3: End-to-End Smoke Tests

Real `claude` in CI, gated to pre-release tags only (`v*-rc*` or equivalent). Locally invoked via an `e2e` feature flag, skipped without `ANTHROPIC_API_KEY`. Two fixture PRDs minimum:

- A happy-path single-task PRD.
- An iteration-recovery PRD (failure on iteration 1, success on iteration 2).

### 14.4 Coverage

70% line-coverage soft floor reports informationally (warning, not hard fail). Revisited once v1 stabilizes.

---

## 15. PRD Partitioning

The v1 implementation is partitioned into nine PRDs:

| PRD                                           | Scope                                                                                                                                                                                                                                                                                   |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **PRD-01: Strict PRD Grammar & Parser**       | The markdown grammar (including the optional `**Category**` metadata key, freeform string), validation rules, error reporting. Validator takes the **repo root** as input so it can resolve `**Skills**` references against `.zurdo/skills/`.                                                                                                                 |
| **PRD-02: Config & Pre-flight Validation**    | `.zurdo/config.toml` schema (including `[defaults]` with `max_attempts` and `max_total_iterations`, required `[roles.analyzer]`), `zurdo init` (config write + bulk skill install), `zurdo init --sync`, pre-flight binary checks, lock acquisition, analyzer-presence check when `--analyze` is set, **interactive resume prompt orchestration (3-way R/X/A on TTY; `--resume`/`--reset`/`--no-prompt` bypass; non-TTY implicit-resume)**.                               |
| **PRD-03: State Management (`prd.json`)**     | Schema (including per-iteration `model`, `tokens_in`, `tokens_out`, `cost_usd_est` fields; nullable when unavailable), atomic writes, `prd_hash` validation, `--reset` semantics (archive `.zurdo/<slug>/` → `.zurdo/<slug>/.archive/<ts>/`). `schema_version` recorded but no migration framework: mismatch is fatal in v1.                                                                                                                                                 |
| **PRD-04: Sequential Task Runner**            | Topo-sort, status state machine, pre-flight criteria check (iteration 0), iteration loop, all-manual short-circuit, Ctrl-C handling, **global iteration budget tracking (`--max-iterations` / `max_total_iterations`, exit code 6, Ctrl-C-style graceful trip behavior per §5.6)**.                                                                                                                                                    |
| **PRD-05: Criteria Execution**                | All five hint types, AND'ing, timeouts, working-dir guarantees.                                                                                                                                                                                                                         |
| **PRD-06: Provider Layer & Agent Invocation** | `AgentCli` / `CompletionCli` traits (including `model`/`tokens_in`/`tokens_out` on the outcome structs), Anthropic and OpenAI Codex implementations (with default CLI flags from §7.6, **requesting structured/JSON output for token extraction**), prompt template (including prior-iteration narrative tail), per-provider effort→model resolution, transient/permanent error classification per adapter, per-iteration stdout/stderr capture to `.zurdo/<slug>/iterations/`, **per-adapter token-count parser and the hardcoded pricing table (§7.8)**. **Also ships `MockAgentCli` and shared test infrastructure** (mock first, real adapters second; Anthropic before Codex within real adapters). |
| **PRD-07: Progress Log & Resume**             | `progress.log` JSONL format (closed event enum, including per-iteration file path/byte fields on `agent_completed`), append semantics, `.zurdo/<slug>/iterations/` directory layout, resume reconciliation algorithm (steps 1–5 of §6.2; the **interactive prompt orchestration in step 0 lives in PRD-02**), crash recovery without VCS, orphan iteration-file handling.        |
| **PRD-08: Skills System**                     | Skill file format (Anthropic-compatible at `.zurdo/skills/<name>/SKILL.md`), parse-time validation of `**Skills**` references, `install_skills` adapter method invoked at two triggers (bulk via `zurdo init`/PRD-02, runtime auto-install via PRD-04 pre-flight), Anthropic copy-to-`.claude/skills/` with `.zurdo-managed` sentinel + collision handling, bundled-skill `list`/`install` subcommands (the latter also performs immediate destination install). |
| **PRD-09: Reporting**                         | Curated report schema (with `report_schema_version`, per-task and per-run token/cost aggregates including `partial: true` flag when any iteration has `cost_usd_est: null`), JSON and markdown serializers from the shared schema, `zurdo report` subcommand, auto-write to `.zurdo/<slug>/reports/`. Deterministic iteration narratives (no LLM). **Optional category-grouped supplementary view when at least one task declares `**Category**`.**                                                             |
| **PRD-10: CLI Surface, Analyze & Run UX**     | Subcommands (`init`, `run`, `report`, `validate`, `skills list/install`); `--analyze` orchestration (deterministic + `analyzer` role invocation, **leading stats banner including effort distribution, average criteria/task, skills referenced, and category breakdown when present**, per-finding static suggestion table, optional LLM elaboration per warning, three-tier color-coded verdict block); **`--analyze --fix` iterative refinement loop (§9.7: default cap=5; 3-iteration anti-thrash window; preservation block locking task IDs / declaration order / `Effort` / `Depends-on`; per-warning trajectory tracking with `[NEW]`/`[PERSISTS]`/`[RESOLVED]` status tags; sibling-file output `<prd>.proposed.md`; final-approval `y/N` prompt on TTY with `mv` on accept; `--no-prompt` and non-TTY bypass; audit trail under `.zurdo/<slug>/analyze-iterations/` with per-iteration PRD + findings JSON + `summary.json`; thrash diagnostic on exit `7` listing persisted-warning trajectories with static hints; last-good promotion on exit `8` parse failure)**; **run-progress stdout stream (banner, per-task/per-iteration headers, in-place spinner + byte counter on TTY, per-criterion live results, per-iteration token/cost continuation line, end-of-run summary table with iterations and tokens/cost aggregate rows)**; **sticky 2-line dashboard header on cursor-control-capable TTYs (task-status counts + iter/elapsed/tokens/cost; capability-detected via `tput`)**; **next-step suggestion line tailored by exit code (5 → re-run; 6 → raise cap; others → omit)**; **agent stdout/stderr tee (TTY-only live tee dimmed/indented; 10s heartbeats on non-TTY; `--quiet-agent` opt-out)**; **TTY/color detection (auto by isatty; honors `NO_COLOR`; `--no-color` / `--no-progress` flags)**; **flags `--max-iterations`, `--resume`, `--no-prompt`, `--fix`** plus existing `--reset`/`--analyze`/`--format`/log-* flags; **exit codes (including widened `6` covering both run budget and fix-loop cap; new `7` for fix-loop thrash; new `8` for fix-loop parse failure)**; help text; diagnostic-log plumbing (stderr default, ladder mapping; `progress.log` writing belongs to PRD-07). |

### 15.1 Ordering

PRDs 01–06 form the v1 "minimum viable" path. PRDs 07–10 layer on top.

Within the v1 core, the dependency order is roughly: 01 → 02 → 03 → 04, with 05 and 06 unlockable in parallel once 04 has its skeleton. PRD-06's mock infrastructure must land before PRDs 04/05/07/08 can complete their integration tests, so 06's internal ordering is **mock-first**.

### 15.2 Cross-PRD Coupling

- **PRD-04 writes to `progress.log`** via the writer defined in PRD-07. PRD-07 owns the format and writer surface; PRD-04 calls it.
- **PRD-08 extends the `AgentCli` trait** with `install_skills`. The trait itself lives in PRD-06; PRD-08 specifies the method signature, the Anthropic implementation, and the `SkillInstallOutcome` semantics. Order: PRD-06 defines the trait shape including `install_skills`, PRD-08 fills in the behavior. The bulk-install trigger is wired by PRD-02 (`zurdo init`); the runtime auto-install trigger is wired by PRD-04 (pre-flight).
- **PRD-01 needs filesystem access** to validate `**Skills**` references. PRD-01's interface takes a repo root, not just a markdown string.
- **PRD-10's `--analyze` orchestration** calls into PRD-01's deterministic validator and PRD-06's `CompletionCli` for the LLM portion. **`--analyze --fix` (§9.7)** reuses the same `CompletionCli` role and writes per-iteration artifacts to `.zurdo/<slug>/analyze-iterations/` — a new subtree parallel to PRD-03's `iterations/` and PRD-09's `reports/`. The fix loop never reads or writes `prd.json` or `progress.log`, so it is decoupled from PRD-03 and PRD-07 at runtime.

---

## 16. Out of Scope for v1

| Item                                               | Disposition | Reason                                                                                |
| -------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------- |
| GitHub Projects v2                                 | Cut         | Adds `gh` dependency throughout, significant complexity.                              |
| `mcpls` / LSP integration                          | Cut         | Experimental, obscure dependency.                                                     |
| GitHub sync system                                 | Cut         | Defeats the simplicity goal.                                                          |
| Copilot agent backend                              | Cut         | Out of scope for a focused CLI.                                                       |
| GitHub Copilot CLI as a v1 provider                | Cut         | Distinct from OpenAI's `codex` CLI (which ships in v1). Copilot's CLI surface doesn't fit the agent-against-a-task pattern. |
| **All git automation (branches, commits)**         | Cut         | User manages VCS. State lives in `prd.json` + `progress.log`. Reconsider in v1.x.     |
| **PR creation (`--create-prs`, `VcsHostAdapter`)** | Cut         | Bundled with git-automation cut.                                                      |
| **`verifier` role**                                | Cut         | Spec previously listed it without specification; no concrete behavior to ship.        |
| **`reporter` role (LLM narratives)**               | Cut         | Iteration narratives render deterministically from `prd.json`.                        |
| Parallel task execution                            | Deferred    | Sequential only in v1. `--parallel N` is a future feature.                            |
| Auto-merge of dependency branches                  | Cut         | No git in v1.                                                                         |
| Reading GFM checkbox state                         | Cut         | Manual criteria carry no machine signal; Zurdo never modifies the PRD.                |
| Per-PRD config overrides                           | Deferred    | One repo-level config in v1.                                                          |
| User-configurable prompt template                  | Deferred    | Fixed template in v1.                                                                 |
| Per-criterion timeout override syntax              | Deferred    | Global default only.                                                                  |
| `**Working-dir**` task field                       | Cut         | Users compose `cd` in `[shell:]`.                                                     |
| Skill inlining into the prompt                     | Cut         | Native install only; unsupported providers get a warning.                             |
| `metrics.jsonl` streaming                          | Cut         | `prd.json` + `progress.log` + curated report schema cover v1 needs.                   |
| `--accept-changes` partial-preservation            | Cut         | Change-detection per task was unspecified; `--reset` is the only mismatch resolution. |
| State schema migration framework                   | Deferred    | `schema_version` recorded; mismatch fatal in v1.                                      |
| `zurdo migrate` stub command                       | Cut         | Add when there's a real migration.                                                    |
| OpenTelemetry trace export                         | Deferred    | Future addition.                                                                      |
| Retry escalation (effort bump on failure)          | Cut         | Powerful heuristic, adds complexity; opt-in flag if revisited.                        |
| Auto-inferred effort from criterion count/type     | Cut         | Explicit declaration only.                                                            |
| Full-screen TUI mode for run-progress              | Deferred    | v1 stream is line-scrolling with a sticky 2-line header + in-place spinner; no alternate-screen or panes. |
| User-overridable `--analyze` suggestion table      | Cut         | Static suggestions are hardcoded; LLM elaboration is the customization path.          |
| User-configurable pricing (`[pricing.<model>]` blocks) | Deferred | Hardcoded table in v1 (§7.8). Override block for self-hosted/discounted pricing is a v1.x addition once the v1 table proves stable. |
| `zurdo report --analyze` (fix-loop reporting via `report`) | Deferred | Fix-loop tokens/cost and per-warning trajectory surface in the §9.7.5 stdout stream and `.zurdo/<slug>/analyze-iterations/summary.json`. `zurdo report` stays a run-only surface in v1; the trajectory data in `summary.json` makes a future `--analyze` reporting mode straightforward to add. |
| Configurable fix-loop thrash window | Deferred | Hardcoded to 3 iterations in v1 (§9.7.4). Widening to a config knob (`[defaults] fix_thrash_window = N`) waits until real-world thrash patterns inform the right default. |
| LLM-driven auto-fix of `--analyze` *errors* (not warnings) | Cut | `--analyze --fix` is warnings-only by design (§9.7.2). Errors are mechanical one-line edits; auto-fixing structural issues like dependency cycles via LLM is a class of risk v1 declines. |
| Per-iteration approval prompt during `--analyze --fix` | Cut | Final approval only (§9.7.4). Per-iteration `y/N` defeats the loop's automation purpose; reviewers can inspect intermediate state via `.zurdo/<slug>/analyze-iterations/<n>.md` after the fact. |
| Retry-on-parse-failure during `--analyze --fix` | Cut | Halt with last-good promotion (§9.7.4). Feeding parse errors back to the LLM risks consuming the iteration budget on parse recovery rather than warning fixing. |
| In-place PRD overwrite during `--analyze --fix` (no sibling file) | Cut | Sibling-file output is the default and only mode (§9.7.4). Users who want in-place overwrite run `mv <prd>.proposed.md <prd>.md` after the loop — the same operation Zurdo performs on `y` confirmation. |
| `--category <name>` runtime filter                 | Deferred    | `**Category**` is metadata-only in v1 (used by report grouping and analyze stats). A filter that scopes a run to one category — with the matching dep-graph semantics for cross-category dependencies — is a v1.x addition. |
