# State Management (`prd.json`)

**Overview:** Own the on-disk terminal-state representation for a single Zurdo run: the `prd.json` schema (spec §5.2), atomic load/save with crash-safe writes, `prd_hash` and `schema_version` integrity gates, and the `--reset` archive semantics that move `.zurdo/<slug>/` aside into `.zurdo/<slug>/.archive/<UTC-timestamp>/`. This PRD does not implement the in-flight `progress.log` event stream (PRD-07) or the task runner that mutates state per iteration (PRD-04). It also does not implement skill auto-install (PRD-08) or criteria/agent execution (PRD-05/06). The slug-derivation utility introduced in PRD-02 is reused here, not redefined. Output: a state module that PRD-04 can call to load existing state, mutate it through a typed API, and save it atomically — plus the archive routine PRD-02's resume prompt and `--reset` flag dispatch into.

## Task: State Schema Types
**Category**: Backend
**Priority**: 1

Define the `prd.json` schema in Rust types using `serde`, exactly as specified in spec §5.2. The root is a `RunState` struct containing `schema_version: u32` (literal `1` for v1), `prd_path: String` (repo-relative), `prd_hash: String` (sha1 of the PRD file at last successful parse), `started_at: DateTime<Utc>`, `last_updated: DateTime<Utc>`, and `tasks: BTreeMap<TaskId, TaskState>` (ordered for stable JSON output). Each `TaskState` carries `status: TaskStatus` (the full §4.1 enum: `Pending | Blocked | InProgress | Passed | PassedPendingReview | Failed | BlockedByDependency`), `attempts: u32`, optional `passed_at: Option<DateTime<Utc>>`, and `iterations: Vec<Iteration>`. Each `Iteration` mirrors §5.2 verbatim: `attempt: u32`, `started_at`, `ended_at`, `model: String`, `agent_exit_code: i32`, `agent_stdout_path`, `agent_stderr_path`, `agent_stdout_bytes: u64`, `agent_stderr_bytes: u64`, `tokens_in: Option<u64>`, `tokens_out: Option<u64>`, `cost_usd_est: Option<f64>`, `timed_out: bool` (defaults `false`), and `criteria_results: Vec<CriterionResult>`. The pricing/token-count computation is PRD-06's responsibility; this task only defines the fields so they round-trip cleanly through serde. Add `chrono` (with the `serde` feature) as a new dependency. This task ships types and serde round-trip tests only — no I/O, no atomic writes.

### Acceptance Criteria
- `Cargo.toml` adds `chrono` with the `serde` feature
- A new module `src/state.rs` defines public `RunState`, `TaskState`, `Iteration`, `CriterionResult`, and `TaskStatus` types with serde derives
- `TaskStatus` is a `#[serde(rename_all = "kebab-case")]` enum so `passed-pending-review` and `blocked-by-dependency` serialize as the spec writes them
- The full §5.2 example JSON deserializes into a `RunState` and re-serializes byte-identical (modulo BTreeMap key ordering, which must be sorted)
- Field defaults: `attempts = 0`, `timed_out = false`, `tokens_in = None`, `tokens_out = None`, `cost_usd_est = None`, `passed_at = None`, `iterations = []`
- A `RunState::new(prd_path: String, prd_hash: String, task_ids: impl Iterator<Item = TaskId>) -> Self` constructor returns a fresh state with `schema_version: 1`, current UTC timestamps, and every task seeded at `Pending` with zero iterations
- Unit tests cover: serde round-trip on the §5.2 fixture, default-value serialization (a fresh `RunState` for two tasks), every `TaskStatus` variant's serde name, optional fields round-tripping as `null`
- A fixture file `tests/fixtures/state/example.json` carries the §5.2 example and is the basis for the round-trip test
- The module exposes no I/O — no `load`, no `save`, no filesystem types
- Running `cargo test` passes
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly

## Task: State Directory Layout and Slug Reuse
**Category**: Backend
**Priority**: 2

Centralize the `.zurdo/<slug>/` directory layout. PRD-02 introduced the slug-derivation function (`<basename>-<hash4>` where `hash4 = sha1(repo-relative-path)[..4]`); this task imports it (no redefinition) and adds path-builder helpers for every subpath the runtime needs: `prd_json_path(repo_root, slug)`, `progress_log_path(...)`, `iterations_dir(...)`, `reports_dir(...)`, `archive_dir(...)`, and `lock_path(...)`. The helpers are pure functions returning `PathBuf`. This task also adds a `StateDir` newtype that wraps the absolute path to `.zurdo/<slug>/` and ensures the directory (and `iterations/`) exists via `create_dir_all` on first use. The intent is that PRD-04, PRD-07, and PRD-09 all reach for the same path helpers and never reconstruct the layout ad hoc.

### Acceptance Criteria
- A new `src/state/layout.rs` (or `src/state_layout.rs`) exposes pure-function path helpers: `prd_json_path`, `progress_log_path`, `iterations_dir`, `reports_dir`, `archive_dir`, `lock_path`
- Each helper takes `repo_root: &Path` and `slug: &str` and returns `PathBuf`
- `StateDir::ensure(repo_root: &Path, slug: &str) -> io::Result<StateDir>` creates `.zurdo/<slug>/` and `.zurdo/<slug>/iterations/` if they do not exist, and returns a newtype carrying the absolute path
- `StateDir` exposes accessor methods mirroring the path helpers so call sites can write `state_dir.prd_json()` rather than `prd_json_path(repo_root, slug)`
- The slug-derivation function is the one defined by PRD-02 — this task imports and re-exports it, but does NOT redefine the hashing logic
- Path-builder unit tests assert that all six helpers compose the expected `.zurdo/<slug>/...` paths on a fixed repo root
- A `StateDir::ensure` integration test creates a temp directory, calls the function, and asserts `.zurdo/<slug>/iterations/` exists on disk
- Running `cargo test` passes
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly

## Task: Atomic Load and Save Lifecycle
**Category**: Backend
**Priority**: 3

Implement crash-safe persistence of `RunState`. The `save` operation writes to a sibling temp file (`prd.json.tmp.<pid>.<nanos>`) and `rename`s it onto `prd.json` so partially-written files never appear on disk — this is the "atomic writes via temp-file-and-rename" invariant from spec §5.2. The `load` operation reads `prd.json`, parses it via serde, and returns either a `RunState` or a `LoadError` (file-missing, parse error, schema-version mismatch — the schema check itself is the next task; this task surfaces the field but does not branch on it). Both operations update `last_updated` automatically on save. Add a `RunStateStore` struct that owns a `StateDir` plus a cached `RunState` and exposes `load_or_init`, `save`, and a `mutate(|state: &mut RunState| { ... })` helper that takes a closure, runs it, updates `last_updated`, and atomically persists. Concurrent writers within one process are the caller's problem (PRD-02's lock guarantees single-process). Concurrent readers see only fully-committed state thanks to the rename.

### Acceptance Criteria
- A new `src/state/store.rs` (or `src/state_store.rs`) exposes a `RunStateStore` struct
- `RunStateStore::new(state_dir: StateDir) -> Self` constructs an empty store (no I/O)
- `RunStateStore::load(&mut self) -> Result<&RunState, LoadError>` reads `prd.json` from the state dir; on a missing file returns `LoadError::Missing`
- `RunStateStore::save(&self, state: &RunState) -> Result<(), SaveError>` writes the JSON to a temp file in the same directory and `rename`s it onto `prd.json`; the temp filename embeds the PID and a high-resolution timestamp to avoid collisions
- `RunStateStore::mutate<F>(&mut self, f: F) -> Result<(), SaveError>` where `F: FnOnce(&mut RunState)` runs `f` against the cached state, updates `last_updated` to the current UTC time, and persists atomically
- Killing the process between the temp-file write and the rename leaves `prd.json` unchanged (verifiable by writing a corrupted temp file by hand and asserting `load` still parses the original)
- JSON output is pretty-printed (two-space indent) for diff-friendliness — tests assert the on-disk bytes contain newlines
- `save` is fsync-safe to the level Rust's stdlib provides (`File::sync_all` on the temp file before rename); document any platform caveats inline with a one-line comment
- Unit tests cover: save-then-load round trip preserves every field, save updates `last_updated`, two saves in a row yield two distinct `last_updated` timestamps, partial temp-file presence does not corrupt a valid `prd.json`
- An integration test in `tests/state_store.rs` exercises a real filesystem temp directory end-to-end
- Running `cargo test` passes
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly

## Task: `prd_hash` and `schema_version` Integrity Checks
**Category**: Backend
**Priority**: 4

Implement the two run-start integrity gates that spec §11.3 step 7 and §5.5 require. The `prd_hash` gate compares the sha1 of the PRD file's current contents against the `prd_hash` stored in `prd.json` from the previous run; a mismatch returns a structured diff describing which task ids were added, removed, or had their content change (content here means a hash of each task's H2 + metadata + Description + Acceptance Criteria block — implementation can rehash per task and compare). The `schema_version` gate asserts the stored `schema_version` equals the compile-time constant `1`; a mismatch is fatal (no migration framework in v1 per spec §16) and returns an error directing the user to `--reset`. Both gates run when `prd.json` is present at run start. When `prd.json` is absent (fresh run), both gates are no-ops and the runner proceeds to seed a new `RunState`. This task ships the gate functions and their unit tests; PRD-04 will be the caller that maps a gate failure to exit code 4 (state mismatch) per spec §12.3.

### Acceptance Criteria
- A new `src/state/integrity.rs` (or `src/state_integrity.rs`) exposes `check_prd_hash(prd_source: &str, stored: &RunState) -> Result<(), HashMismatch>` and `check_schema_version(stored: &RunState) -> Result<(), SchemaVersionMismatch>`
- `check_prd_hash` computes sha1 of `prd_source` and compares to `stored.prd_hash`; on equality returns `Ok(())`
- On mismatch, `HashMismatch` carries `added: Vec<TaskId>`, `removed: Vec<TaskId>`, and `modified: Vec<TaskId>`, populated by re-parsing the new PRD and comparing per-task content hashes against derivable signals in the stored state
- For v1, "modified" can be a coarse signal — the spec only requires the user be told *something* meaningful changed; if implementing per-task content hashes inside `RunState` is out of scope here, document the simplification with a one-line code comment and at minimum populate `added` and `removed` precisely
- `check_schema_version` asserts `stored.schema_version == 1` and returns `SchemaVersionMismatch { found, expected: 1 }` otherwise
- Both errors implement `Display` such that the human-readable message includes the `--reset` recovery hint exactly as spec §5.5 describes
- The functions are pure and take owned/borrowed data only — no filesystem I/O, no `Path` arguments
- Unit tests cover: matching hash passes, mismatching hash with adds-only, mismatching hash with removes-only, mismatching hash with both, schema version `1` passes, schema version `0` and `2` fail with the expected diagnostic
- Running `cargo test` passes
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly

## Task: `--reset` Archive Semantics
**Category**: Backend
**Priority**: 5

Implement the `--reset` archive routine described in spec §5.5. When invoked (either via the `--reset` flag, or via the `[X] Reset` choice from PRD-02's resume prompt), Zurdo moves the entire existing `.zurdo/<slug>/` directory — except `lock`, which the active run still holds — into `.zurdo/<slug>/.archive/<UTC-timestamp>/`, then leaves a fresh empty `.zurdo/<slug>/` directory (with `iterations/`) ready for the new run to seed. Archived state is preserved indefinitely and not garbage-collected; users may delete `.zurdo/<slug>/.archive/` manually at any time. The timestamp format is `YYYY-MM-DDThh-mm-ssZ` (filesystem-safe — colons replaced with hyphens). If the archive subdir already exists for the same second (e.g. two `--reset` invocations within a second), append a `-<u32>` disambiguator. This task ships only the archive routine; wiring it into the CLI's flag dispatch and the resume-prompt's `Reset` outcome is PRD-04's job, but the routine itself must be callable in isolation.

### Acceptance Criteria
- A new `src/state/reset.rs` (or `src/state_reset.rs`) exposes `archive_state(state_dir: &StateDir, now: DateTime<Utc>) -> Result<PathBuf, ArchiveError>` returning the absolute path to the created archive subdirectory
- The routine moves every entry in `.zurdo/<slug>/` into `.zurdo/<slug>/.archive/<timestamp>/` except `lock` (which is left in place — the active run still holds it)
- The destination timestamp is formatted as `YYYY-MM-DDThh-mm-ssZ` with colons replaced by hyphens
- If the destination directory already exists, the routine retries with `-1`, `-2`, ... appended until it finds an unused name; the disambiguator is bounded (e.g. retry up to 1000 times) and bubbles `ArchiveError::DisambiguatorExhausted` past that
- After moving, `.zurdo/<slug>/` exists and is empty except for `lock` and a freshly-recreated `iterations/`
- The routine does not delete the archive after creation — it is preserved indefinitely
- If `.zurdo/<slug>/` is empty before the call (no prior state), the routine is a no-op returning the path of an empty archive directory (still created, for audit symmetry)
- `ArchiveError` variants cover `Io(io::Error)`, `DisambiguatorExhausted`, and `LockMissing` (called when the active lock file is unexpectedly gone — surfaces as a recoverable warning to the caller)
- Unit tests in a temp directory cover: archive of a populated state dir with `prd.json` + `iterations/`, archive of an empty state dir, two archives in the same second collide and disambiguate, the active `lock` file remains in place across the call
- Integration test in `tests/state_reset.rs` drives the routine against a real filesystem temp directory
- Running `cargo test` passes
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly
