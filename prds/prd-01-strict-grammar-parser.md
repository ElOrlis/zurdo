# PRD-01: Strict Grammar Parser

**Overview:** Establish the Rust crate skeleton and implement the markdown PRD parser end-to-end: AST types, hint tokenization, line-based markdown parser, and semantic validator wired to a `zurdo validate` CLI subcommand. Output is a strict, line-numbered diagnostics pipeline that downstream PRDs (config & pre-flight, state management, agent loop) consume. This PRD intentionally stops short of `prd.json` persistence (PRD-03), config loading (PRD-02), and runtime execution (PRD-04). It also defers `effort_map` enum-membership checks to PRD-02, since the parser does not load `.zurdo/config.toml`.

## Task: Bootstrap Rust Project
**Category**: Infrastructure
**Priority**: 1
**Effort**: low
**Depends-on**: []

Set up the Cargo project that all subsequent PRDs build on. Single binary crate named `zurdo` with internal modules for parser, ast, validator, error, and cli. Pin Rust edition 2024. Add baseline dependencies that PRD-01 needs and that downstream PRDs will reuse: `clap` for CLI parsing, `thiserror` for error types, `serde` and `serde_json` for upcoming `prd.json` persistence, `sha1` for the upcoming slug hash, and `regex` for parser internals. Configure `rustfmt` and `clippy` with project-wide settings. The `.gitignore` excludes `target/` and `.zurdo/`. The `main.rs` wires a no-op `validate` subcommand that prints a placeholder message and exits `0`, just to prove the binary builds end to end.

### Acceptance Criteria
- `Cargo.toml` exists at the repo root with `name = "zurdo"` and `edition = "2024"`
- `Cargo.toml` declares dependencies on `clap`, `thiserror`, `serde`, `serde_json`, `sha1`, and `regex`, each pinned with a caret version specifier so minor upgrades are allowed but major upgrades are explicit
- Source files `src/main.rs`, `src/parser.rs`, `src/ast.rs`, `src/validator.rs`, `src/error.rs`, and `src/cli.rs` all exist (modules may be empty stubs)
- A `rustfmt.toml` file exists at the repo root with at least the edition pinned to 2024
- A `clippy.toml` file exists at the repo root with `msrv` pinned to the project's minimum supported Rust version, and `cargo clippy --all-targets -- -D warnings` exits cleanly
- The `.gitignore` file contains both `target/` and `.zurdo/` on their own lines
- Running `cargo build` succeeds with zero warnings
- Running `cargo test` succeeds (zero tests is acceptable at this stage)
- Running `cargo fmt --check` succeeds
- Running `cargo run -- validate dummy.md` exits with code `0` and prints a placeholder message, proving the clap wiring is in place

## Task: AST Types and Hint Parsing
**Category**: Backend
**Priority**: 2
**Effort**: medium
**Depends-on**: [task-1]

Define the strongly-typed in-memory representation of a parsed PRD, plus the standalone hint parser. The AST captures everything the grammar permits: PRD title and intro, a list of Task entries each with metadata, description, and criteria. Metadata uses a newtype `Effort` wrapping a `String` so consumers cannot confuse it with arbitrary strings; the enum-membership check against `effort_map` is PRD-02's job, not this PRD's. The `Hint` type is an enum with five variants covering shell, http (with method, URL, and expected status), file-exists, grep (with pattern and file), and manual. The hint parser is a pure function that takes a criterion line and returns the human-readable text plus a vector of hints in source (left-to-right) order, greedily consuming bracket-delimited blocks from the end of the line. This task ships no markdown parsing and no I/O. It is pure data types and hint tokenization logic, fully unit-tested.

### Acceptance Criteria
- The `src/ast.rs` module defines public structs and enums for `Prd`, `Task`, `TaskMetadata`, `Description`, `Criterion`, `Hint`, `Duration`, `Effort`, and `TaskId`
- The `Effort` type is a newtype wrapping a `String` rather than a bare `String` field on `TaskMetadata`
- The `TaskId` type is a newtype with a constructor that validates against the regex pattern `^task-[a-z0-9-]+$` and returns a `Result` with a `ParseError::InvalidTaskId` on rejection (uppercase letters and non-ASCII characters are rejected)
- `ParseError` is a single `thiserror`-derived enum living in `src/error.rs`, every variant carries a `line: usize` and a per-variant payload, and all parser-side errors in PRD-01 are variants of this one type (no ad-hoc `String` errors)
- The `Hint` enum's `Shell` variant has a single `cmd: String` field
- The `Hint` enum's `Http` variant has `method: String`, `url: String`, and `expected_status: u16` fields
- The `Hint` enum's `FileExists` variant has a single `path: String` field
- The `Hint` enum's `Grep` variant has `pattern: String` and `file: String` fields
- The `Hint` enum's `Manual` variant is a unit variant carrying no data
- The `Duration` type parses values like `30m` to thirty minutes and like `1h` to one hour, but rejects bare integers like `30` with a `ParseError::InvalidDuration`
- A `parse_hints` function lives in the parser module and is callable independently of full PRD parsing
- Calling `parse_hints` on a criterion line containing two bracket hints returns the human text without the brackets plus a vector of two hints in source (left-to-right) order, regardless of the right-to-left consumption strategy used internally
- A criterion line with no bracket hints causes `parse_hints` to return an empty hint vector (the at-least-one-hint rule is enforced by the validator in Priority 4, not by the hint parser)
- Nested brackets inside a hint payload are preserved verbatim: e.g., `[shell:grep "[a-z]" file.txt]` yields a single `Shell` hint whose `cmd` retains the inner `[a-z]` regex intact, and is not split into two hints
- Unit tests cover all five hint types parsed correctly, multiple hints on one line preserved in source order, nested brackets in payloads, malformed hints (empty shell command, http hint missing the expected status) producing a typed `ParseError`, bare integer durations rejected, valid task ids accepted, and invalid task ids (uppercase, empty suffix, non-ASCII) rejected
- Running `cargo test` passes with at least eighteen unit tests in this module
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly

## Task: Markdown Parser
**Category**: Backend
**Priority**: 3
**Effort**: high
**Depends-on**: [task-2]

Implement the hand-rolled line-based parser that turns a markdown PRD file into a `Prd` AST. The parser walks the document line by line, recognizing four kinds of structural anchor: the H1 PRD title (exactly one, required as the first non-blank line), the free-form prose intro (everything until the first H2), H2 task headings using the format `## Task: task-id — title` with a literal em-dash separator (U+2014, not a hyphen-minus), and H3 section headings within a task (only `Description` and `Acceptance Criteria` are allowed, in that order). Metadata fields appear as a contiguous block of bold-key colon-value lines directly under each H2, with no blank lines interrupting the block. The closed key enum is `Effort`, `Depends-on`, `Max-Attempts`, `Skills`, `Agent-timeout`, and `Category`. The `Depends-on` field parses YAML-style arrays like `[task-1, task-2]` or an empty `[]`. Acceptance criteria are GFM task-list items whose trailing bracketed hint blocks are passed to the Priority 2 hint parser. This task produces only structural parsing. Semantic rules such as id uniqueness, dependency resolution, the at-least-one-hint rule, and skill-directory existence are Priority 4's job. Every parser error carries the originating line number via `ParseError` from Priority 2.

### Acceptance Criteria
- The `src/parser.rs` module exposes a public function `parse_prd` that takes the raw markdown contents as a `&str` and returns a `Result<Prd, ParseError>`, with no filesystem access
- The parser is hand-rolled and line-based, and no external markdown AST crate such as `pulldown-cmark` is added to `Cargo.toml`
- A missing or duplicate H1 produces a `ParseError` variant with the offending line number
- An H2 heading that does not match the required task heading format produces an error citing the line number; the parser specifically rejects a hyphen-minus (U+002D) used in place of the required em-dash (U+2014)
- A metadata key not in the closed enum produces an error naming the offending key and line number
- A duplicate metadata key within one task's metadata block produces a `ParseError::DuplicateMetadataKey` naming the key and both line numbers
- A blank line inside the metadata block ends the block, and any later bold-key colon-value lines under the same task are treated as prose and produce a structured error (exact user-facing wording is left to the implementation)
- Missing required metadata keys (`Effort` or `Depends-on`) produces one error per missing key
- An H3 other than `Description` or `Acceptance Criteria`, or those two in the wrong order, produces an error citing the line number
- The `Depends-on` field correctly parses an empty array, a single-element array, and a multi-element array; malformed arrays produce a structured error
- Acceptance criteria are recognized only as GFM task-list items, and the parser does not read or write the checked or unchecked state (Zurdo always treats them as unchecked per the spec)
- Each parsed `Criterion`, `Task`, and metadata field carries the original line number for downstream error reporting
- Files using CRLF (`\r\n`) line endings parse to an identical `Prd` AST as the same file using LF (`\n`) line endings, and a leading UTF-8 BOM is stripped silently
- Trailing whitespace on otherwise-valid metadata and criterion lines is tolerated and does not produce a parse error
- Unit tests cover a minimal valid PRD with one task, PRDs exercising each error class above, and a PRD with two tasks where the second has all optional metadata fields populated
- Fixture files under `tests/fixtures/valid/` and `tests/fixtures/invalid/` exist with at least one fixture per error class, exercised by integration tests under `tests/parser.rs`
- Running `cargo test` passes with all parser unit and integration tests
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly

## Task: Validator and Validate CLI
**Category**: Backend
**Priority**: 4
**Effort**: medium
**Depends-on**: [task-3]

Implement the semantic validator and wire it to the `zurdo validate` subcommand. The validator takes the AST plus a repo-root path so it can resolve `Skills` references against `.zurdo/skills/<name>/SKILL.md` on disk, per spec §11.3 and §8. Semantic rules enforced: task-id uniqueness within the PRD; every `Depends-on` entry refers to a task defined in the same PRD; no self-dependencies; no cycles in the dependency graph; every acceptance criterion carries at least one hint (a criterion with only the manual hint is fine, but a criterion with zero hints is an error); every `Skills` reference resolves to an existing `SKILL.md` file. `Effort` value membership in `effort_map` is explicitly NOT checked here. That is PRD-02's pre-flight job because the parser does not load `.zurdo/config.toml`. The CLI subcommand reads the file, parses it, validates it, and prints diagnostics to stderr (human-readable, line-numbered, one per problem) on failure with exit code `1`, or prints `OK` to stdout with exit code `0` on success.

### Acceptance Criteria
- The `src/validator.rs` module exposes a `validate` function that takes a `&Prd` and a `&Path` repo-root and returns a `Result<(), Vec<ValidationError>>`, so all problems surface in one run rather than just the first
- `ValidationError` is a single `thiserror`-derived enum in `src/error.rs` with variants `DuplicateTaskId`, `UnknownDependency`, `SelfDependency`, `DependencyCycle`, `CriterionMissingHint`, and `UnknownSkill`, each carrying the line number(s) needed for `file:line:message` diagnostics
- Duplicate task ids across H2 headings within one PRD produce a `DuplicateTaskId` error naming the id and the lines on which it appears
- A `Depends-on` entry referencing an unknown task id produces an `UnknownDependency` error naming the task and the missing reference
- A task that lists itself in `Depends-on` produces a `SelfDependency` error
- A dependency cycle (two-task or longer) produces a `DependencyCycle` error naming all tasks in the cycle in traversal order
- A criterion whose parsed hint list is empty produces a `CriterionMissingHint` error naming the task and the line number
- A criterion carrying only `[manual]` and no other hints passes validation, and a fixture under `tests/fixtures/valid/` exercises this case alongside a paired fixture under `tests/fixtures/invalid/` whose criterion has zero hints
- Multiple errors of the same class are each reported once per occurrence (no deduplication across distinct lines)
- A `Skills` reference where the corresponding `SKILL.md` file does not exist on disk produces an `UnknownSkill` error naming the task, the skill name, and the expected path
- The validator does NOT check `Effort` values against any enum, because effort-map validation is deferred to PRD-02 and is explicitly out of scope for this PRD
- A task with no `Depends-on` entries and no incoming dependency edges is valid (orphan/isolated tasks are not an error)
- Running `cargo run -- validate <path>` on a valid PRD exits with code `0` and prints `OK` to stdout
- Running `cargo run -- validate <path>` on an invalid PRD exits with code `1` and prints one human-readable diagnostic per error to stderr, each in the form `file:line:message` so editors can jump to it
- A missing PRD file produces an exit code of `1` and a clear file-not-found error on stderr, rather than a panic
- An end-to-end test in `tests/validate_cli.rs` runs the binary against fixture PRDs (valid plus each invalid class) via the `assert_cmd` crate (added to dev-dependencies) and asserts both the exit code and the stderr content
- An integration smoke test parses, validates, and exits `0` on a multi-task fixture that exercises the full pipeline (parser → AST → hint parser → validator → CLI), proving Priorities 2, 3, and 4 compose end to end rather than only passing in isolation
- Fixture PRDs cover at minimum: valid single-task, valid multi-task with dependencies, duplicate ids, unknown dependency, self-dependency, two-task cycle, three-task cycle, criterion missing hints, criterion with only `[manual]` (valid), and unknown skill reference
- The skill resolution test verifies that a `Skills` reference whose `SKILL.md` exists on disk passes and one whose file is absent fails with `UnknownSkill` (implementation may use a temporary directory or static fixtures, at the implementer's discretion)
- Running `cargo test` passes for all unit, integration, and CLI tests
- Running `cargo clippy --all-targets -- -D warnings` exits cleanly
- Running `cargo fmt --check` passes
