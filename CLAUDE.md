# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository status

This repo is **pre-implementation**. There is no Rust source, no `Cargo.toml`, and no build/test/lint commands yet. The entire repo currently consists of design documents under `specs/` (and an empty `prds/` directory). Do not invent build commands or claim things compile until the Rust project actually exists.

## What Zurdo is

Zurdo is a planned **Rust rewrite of [paullovvik/ralph-loop](https://github.com/paullovvik/ralph-loop)**. It is a CLI that drives LLM agents through a PRD's tasks in a loop: parse a markdown PRD, topo-sort tasks by dependencies, prompt an agent CLI (`claude`, `gh copilot`) per task, then independently verify acceptance criteria via inline type hints (`[shell:]`, `[http:]`, `[file-exists:]`, `[grep:]`, `[manual]`). It shells out to provider CLIs via `std::process::Command` — no SDKs, no API calls.

Intentionally excluded: GitHub Projects v2, mcpls/LSP, Copilot agent backend, GitHub sync.

## Architecture orientation

Read these in order before touching anything substantive:

1. `specs/zurdo-spec.md` — the **feature spec**. Defines the 9 features (dep graph, `.zurdo/` state dir, independent verification, per-task git branches/PRs, per-task retry, effort→model mapping, multi-provider, skills, monitoring) and ends with open questions.
2. `specs/zurdo-spec-resolutions.md` — the **interview-resolved decisions** that supersede the open questions in the spec. This is the load-bearing document for partitioning into PRDs: it pins down the PRD grammar, hint semantics, task-status enum, runtime semantics (cumulative branching, retry, escalation), provider abstraction, and config layout. When the spec and the resolutions disagree, **resolutions win**.
3. `specs/architecture.md` — Mermaid block-diagram of the runtime: Input → Core (parser → validator → dep graph → scheduler → skill loader → prompt builder → agent loop with verifier/retry/escalator) → Providers → Git layer → `.zurdo/` state → Output.

Key invariants the design hinges on (so future code stays consistent):

- **State lives at `.zurdo/<basename>-<hash4>/`** at repo root, never beside the PRD. Slug is deterministic: `sha1(repo-relative path)[0..4]`.
- **Zurdo never modifies the PRD file** — GFM checkboxes are never read or written; manual criteria carry zero machine signal.
- **Task IDs match `^task-[a-z0-9-]+$`**, unique within a PRD. Dependencies are local to one PRD; cross-PRD deps are validation errors.
- **Every acceptance criterion must carry at least one type hint**; multiple hints on one criterion are AND'd; `[manual]` mixed with automated hints means automated still gates.
- **Effort is a closed enum defined by `effort_map` in `.zurdo/config.toml`** — the grammar does *not* hardcode `low|medium|high`.
- **A dep is satisfied at `passed` or `passed-pending-review`** (manual review is out-of-band).
- **Provider abstraction is a thin trait** (`invoke(prompt) -> Result<String>`) — mirror the original ralph-loop's shell-out simplicity. Don't over-abstract.

## Working on the specs

- The `prds/` directory is currently empty; per-feature PRDs are expected to be partitioned out of `zurdo-spec-resolutions.md`. When generating PRDs, follow the strict grammar specified in §1.1 of that document (H2 `## Task: <id> — <title>`, contiguous metadata block, required `Effort` and `Depends-on`, criteria as `- [ ]` with mandatory hints).
- When editing `specs/architecture.md`, preserve the Mermaid `flowchart TD` structure and the existing subgraph groupings (INPUT / CORE / PROVIDERS / ROLES / CHECKS / GIT / STATE / OUTPUT).
