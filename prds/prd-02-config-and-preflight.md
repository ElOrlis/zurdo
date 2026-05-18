# Config and Pre-flight Validation

**Overview:** Build the second layer on top of PRD-01: load and validate `.zurdo/config.toml`, wire up the deferred `effort_map` membership check, ship the `zurdo init` and `zurdo init --sync` subcommands, and implement the run-start pre-flight pipeline that gates every `zurdo run` invocation (config presence, role/provider consistency, CLI binary presence on `PATH`, run lock acquisition, analyzer-role presence when `--analyze`, and the three-way interactive resume prompt). This PRD owns config-driven behavior end to end but stops short of `prd.json` persistence (PRD-03), the task runner (PRD-04), and criteria/agent execution (PRD-05/06). Skill installation (`init` bulk install, runtime auto-install) is invoked here but the install machinery itself lives in PRD-08; this PRD calls a trait method whose default implementation may be a no-op stub until PRD-08 lands.

## Task: Config Schema and TOML Loader
**Category**: Backend
**Priority**: 1

Define strongly-typed structs for `.zurdo/config.toml` and load them via `toml` (added as a new dependency). The schema mirrors spec §11.2: `[roles.executor]` (required, with `provider`), `[roles.analyzer]` (required when used, with `provider` and `model`), `[effort_map.<provider>]` blocks keyed by effort name to model id, `[defaults]` with `max_attempts` (default 5) and `max_total_iterations` (default 0 = unlimited), `[timeouts]` with `criterion_seconds` (default 300) and `agent_seconds` (default 1800), and `[providers.<name>]` with `cli` and `extra_args`. Unknown top-level tables and unknown keys within known tables are errors. A missing `.zurdo/config.toml` is an error with a hint pointing at `zurdo init`. This task ships only the loader and its tests — no CLI subcommand wiring, no pre-flight, no init. The loader is a pure function from a file path to a `Result<Config, ConfigError>`.

### Acceptance Criteria
- `Cargo.toml` adds `toml` as a dependency
- A new module `src/config.rs` exposes a `Config` struct with public fields matching spec §11.2 exactly, plus a `load(path: &Path) -> Result<Config, ConfigError>` function
- A new `ConfigError` enum lives in `src/error.rs` (or a sibling module) with variants for file-not-found, parse-failure, missing-required-table, unknown-table, unknown-key, missing-required-key, and provider-not-defined
- A missing `.zurdo/config.toml` returns a file-not-found error whose message includes the hint `run \`zurdo init\` to create one`
- A config that omits `[roles.executor]` returns a missing-required-table error naming `roles.executor`
- A config whose `roles.executor.provider = "anthropic"` but has no `[providers.anthropic]` block returns a provider-not-defined error naming `anthropic`
- A config containing an unknown top-level table such as `[roles.verifier]` returns an unknown-table error naming the offending table and its source line
- A config with an unknown key inside a known table (e.g. `[defaults] foo = 1`) returns an unknown-key error naming the offending key and table
- Defaults are applied when `[defaults]`, `[timeouts]`, or `[providers.*].extra_args` are omitted: `max_attempts = 5`, `max_total_iterations = 0`, `criterion_seconds = 300`, `agent_seconds = 1800`, `extra_args = []`
- Unit tests cover one full happy-path fixture, plus a negative fixture per error variant above
- Fixture configs live under `tests/fixtures/config/valid/` and `tests/fixtures/config/invalid/`
- Running `cargo test` passes
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly

## Task: Effort-Map Membership Validation
**Category**: Backend
**Priority**: 2

Wire the deferred `effort_map` check from PRD-01 into a new pre-flight function. PRD-01's parser intentionally accepts any string for `**Effort**` because it does not load config; once config is available, every task's `Effort` value must be a key in `[effort_map.<roles.executor.provider>]`. Implement `validate_effort_map(prd: &Prd, config: &Config) -> Result<(), Vec<ValidationError>>` in a new `src/preflight.rs` module. The function looks up the executor provider, fetches the corresponding effort map, and asserts every task's `Effort` value is a key in that map. A missing entry yields a `UnknownEffort` error naming the task, the offending value, and the list of valid keys (so users see exactly what they can type). This validation runs separately from PRD-01's structural validator so unit-testable in isolation.

### Acceptance Criteria
- A new `src/preflight.rs` module exposes `validate_effort_map(prd: &Prd, config: &Config) -> Result<(), Vec<ValidationError>>`
- `ValidationError` (already defined by PRD-01) gains an `UnknownEffort { task_id, value, valid_keys }` variant
- A task whose `Effort` value matches a key in the executor's effort map passes the check
- A task whose `Effort` value is missing from the executor's effort map fails with `UnknownEffort` listing the valid keys in sorted order
- If `[effort_map.<executor.provider>]` itself is missing from config, every task fails with one `UnknownEffort` each (no special "map missing" path; the empty key set communicates the same fact)
- Unit tests cover at least: happy path (all tasks valid), one task invalid, multiple tasks invalid, missing effort-map block
- The check is callable independently — no filesystem I/O, no CLI plumbing, no run-time state
- Running `cargo test` passes
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly

## Task: `zurdo init` and `--sync` Subcommands
**Category**: Backend
**Priority**: 3

Implement `zurdo init` and `zurdo init --sync` per spec §11.2 and §15. `zurdo init` writes a commented default `.zurdo/config.toml` to the repo root if one is not already present (it errors with a hint if one exists, unless `--force` is passed — even then it confirms on TTY). It also creates `.zurdo/skills/` if absent and triggers the bulk skill-install pathway. `zurdo init --sync` re-runs the install pathway without rewriting `config.toml`, so users adding a new skill don't need a full init. The install pathway delegates to `AgentCli::install_skills`, a trait method that PRD-08 will implement; for this PRD a stub trait with a no-op default implementation is sufficient, since the install behavior is exercised end-to-end in PRD-08's acceptance criteria. The default config contains both `[effort_map.anthropic]` and `[effort_map.codex]` blocks plus both `[providers.*]` blocks even if only one is referenced, so switching executor is a one-line change.

### Acceptance Criteria
- `src/cli.rs` (or a sibling module) wires `init` and `init --sync` subcommands via clap
- Running `zurdo init` in a fresh repo creates `.zurdo/config.toml` with a complete default schema (all sections from spec §11.2) and inline comments
- The default config sets `[roles.executor] provider = "anthropic"` and `[roles.analyzer] provider = "anthropic"` with `model = "claude-haiku-4-5"`
- The default config writes both `[effort_map.anthropic]` and `[effort_map.codex]` blocks and both `[providers.*]` blocks
- Running `zurdo init` when `.zurdo/config.toml` already exists exits with code `1` and an error directing the user to `--force` or to edit the existing file
- Running `zurdo init --force` overwrites an existing config on non-TTY stdin; on TTY it prompts `Overwrite existing config? [y/N]` and aborts on anything but `y`/`Y`
- Running `zurdo init` creates `.zurdo/skills/` if missing, and is a no-op for that directory if present
- Running `zurdo init --sync` does not modify `.zurdo/config.toml` even if defaults have drifted; it only re-runs the skill-install pathway
- An `AgentCli` trait stub lives in `src/provider.rs` (or equivalent) with at least an `install_skills` method whose default implementation is a no-op returning `Ok(())`, so PRD-08 can fill in the real behavior without changing this PRD's wiring
- The init pathway logs to stderr at info level: which files were created, which were skipped, and whether the install pathway was invoked
- Integration test in `tests/init_cli.rs` runs the binary in a temp directory and asserts: fresh-init creates expected files, second init errors, `--force` overwrites on non-TTY, `--sync` leaves config untouched
- Running `cargo test` passes
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly

## Task: Pre-flight Pipeline
**Category**: Backend
**Priority**: 4

Implement the pre-flight pipeline that runs at the top of every `zurdo run` invocation, executing checks 1–5, 8, 9 from spec §11.3 (checks 6 and 7 — resume prompt and `prd_hash` — are owned by the next task in this PRD and by PRD-03 respectively; this task wires the call sites so the order in §11.3 is preserved). The pipeline takes a repo-root path, the parsed PRD, and the loaded config, and either returns `Ok(PreflightOutcome)` or `Err(PreflightError)`. It checks: (1) config presence — already done by the loader, but re-asserted at the pipeline boundary; (2) every role's provider exists in `[providers.*]`; (3) every referenced provider's `cli` binary resolves on `PATH` via `which`; (4) the PRD parses cleanly (re-validates by delegating to PRD-01); (5) the `.zurdo/<slug>/lock` file is absent or stale (where a stale lock has a PID that is no longer alive — Unix-only `kill -0` semantics suffice for v1); (8) every `**Skills**` reference is satisfied by either an existing destination or an installable source (resolves to a path check on `.zurdo/skills/<name>/SKILL.md`); (9) when `--analyze` is set, `[roles.analyzer]` is present in config. Each failure yields a distinct error variant so callers (and exit-code mappers in PRD-10) can branch.

### Acceptance Criteria
- A new `src/preflight.rs` adds a `run_preflight(repo_root: &Path, prd: &Prd, config: &Config, opts: PreflightOptions) -> Result<PreflightOutcome, PreflightError>` function
- `PreflightOptions` carries an `analyze: bool` flag (and any other run-mode flags this PRD's pipeline observes)
- `PreflightError` variants exist for `ConfigMissing`, `ProviderNotConfigured`, `CliBinaryMissing { provider, cli, hint }`, `PrdValidationFailed(Vec<ValidationError>)`, `LockHeld { pid, started_at }`, `LockStale { pid }` (info-level, recoverable), `SkillUnresolved`, and `AnalyzerRoleMissing`
- Lock acquisition writes `.zurdo/<slug>/lock` containing `pid + "\n" + started_at_iso8601` and is released on `Drop` of a `RunLock` guard; in v1 this can be a best-effort cleanup with the resume reconciliation in PRD-07 handling crashed leftovers
- A stale lock (PID not alive per `kill -0`) logs a `warn`, overwrites the file, and proceeds — does not return an error
- A held lock (PID alive) returns `LockHeld` with the recorded PID and start time so the CLI can render `another zurdo run is active (pid=N, started=T)`
- A missing CLI binary returns `CliBinaryMissing` with a hint string pointing at the role config (e.g. `[roles.executor] provider = "anthropic" → providers.anthropic.cli = "claude"`) and the canonical install URL for that provider
- When `analyze = true` and `[roles.analyzer]` is absent, returns `AnalyzerRoleMissing` and the pipeline does not fall through to lock acquisition
- A `PreflightOutcome` struct carries the resolved slug (`<basename>-<hash4>`), the absolute path to `.zurdo/<slug>/`, and the acquired `RunLock` guard
- The slug derivation lives in this PRD (mirroring spec §2.1 — `sha1(repo-relative path)[0..4]`) — PRD-03 will reuse the same function from here
- Unit tests cover slug determinism (same path → same slug), and each `PreflightError` variant via fakes (a fake `which` resolver, a temp lock file, etc.)
- Integration test in `tests/preflight.rs` exercises a real binary lookup using a known-present tool (e.g. `cargo`) standing in for the provider CLI
- Running `cargo test` passes
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly

## Task: Interactive Resume Prompt
**Category**: Backend
**Priority**: 5

Implement the three-way interactive resume prompt described in spec §6.2 step 0. When `.zurdo/<slug>/prd.json` already exists at run start, Zurdo prompts the user with three choices: `[R] Resume` (re-use existing state), `[X] Reset` (archive and start over per §5.5 — actual archive logic ships in PRD-03; this PRD just signals the intent), and `[A] Abort` (exit immediately). The prompt is bypassed under any of: `--resume` (forces Resume), `--reset` (forces Reset), `--no-prompt` (forces Resume — CI-safe), or a non-TTY stdin (forces Resume — implicit). Reading and writing `prd.json` itself is PRD-03's responsibility; this task only detects the file's presence, dispatches the prompt, and emits a `ResumeIntent` enum (`Resume | Reset | Abort`) that the runner consumes. On `Abort` the process exits cleanly with code `0`. The prompt loops on invalid input until a valid choice is entered.

### Acceptance Criteria
- A new `src/resume_prompt.rs` module exposes a `prompt_resume(state_dir: &Path, flags: ResumeFlags, tty: &mut dyn Tty) -> ResumeIntent` function
- `ResumeIntent` is a public enum with `Resume`, `Reset`, and `Abort` variants
- `ResumeFlags` captures `resume: bool`, `reset: bool`, `no_prompt: bool`, plus an `is_tty: bool` reflecting whether stdin is a terminal
- The `Tty` trait wraps stdin reads and stdout writes so the prompt can be unit-tested with an in-memory fake
- When `prd.json` does not exist at `state_dir`, the function returns `Resume` immediately without prompting (no-op for fresh runs)
- When `--reset` is set, returns `Reset` without prompting regardless of TTY state
- When `--resume` is set, returns `Resume` without prompting regardless of TTY state
- When `--no-prompt` is set without `--resume`/`--reset`, returns `Resume` without prompting (documented bypass)
- When stdin is non-TTY and none of the bypass flags are set, returns `Resume` and logs an info-level note explaining the implicit choice
- When stdin is a TTY and no bypass flag is set, prompts `[R] Resume / [X] Reset / [A] Abort:` and returns the matching intent
- The prompt is case-insensitive (accepts `r`, `R`, `x`, `X`, `a`, `A`) and rejects anything else with `Invalid choice — type R, X, or A` and re-prompts
- On `Abort`, the caller is expected to exit `0`; this task does not call `std::process::exit` itself
- `--resume` and `--reset` together return an error before the prompt runs (caller surfaces a clap-level conflict; document the conflict in the flag help text)
- Unit tests cover every bypass path, the TTY happy path (one valid keypress), the TTY re-prompt path (invalid then valid), and the no-state-yet shortcut
- Integration test in `tests/resume_prompt.rs` drives the prompt with a fake TTY and asserts the returned intent
- Running `cargo test` passes
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly
