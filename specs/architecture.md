## Runtime overview

```mermaid
flowchart TD
    subgraph INPUT["Input Layer"]
        PRD["PRD File\n.md"]
        CONFIG[".zurdo/config.toml"]
        SKILLS[".zurdo/skills/<name>/"]
    end

    subgraph CORE["Zurdo Core (Rust)"]
        CLI["CLI Entry Point\nclap"]
        PARSER["PRD Parser\nstrict grammar"]
        VALIDATOR["Validator\nSchema · Dep Cycles · Effort · Skills"]
        DEPGRAPH["Dependency Graph\nTopological Sort"]
        SCHEDULER["Task Scheduler\ndeclaration-order tiebreaker"]
        SKILLINSTALLER["Skill Installer\n.zurdo/skills → .claude/skills"]
        PROMPTBUILDER["Prompt Builder\nTask + Available Skills + Prev Feedback"]
        LOCK["Lock Manager\n.zurdo/<slug>/lock"]

        subgraph LOOP["Run Loop (per task, sequential)"]
            PREFLIGHT["Pre-flight Verifier\niteration 0"]
            AGENT["Agent Invoker\nstd::process::Command"]
            CLASSIFIER["Error Classifier\nTransient · Permanent · Success"]
            VERIFIER["Independent Verifier\ntokio parallel"]
            RETRY["Attempt Tracker\nper-task Max-Attempts"]
        end

        FIXLOOPBOX["Fix Loop\n--analyze --fix\n(see detail below)"]

        STATEMGR["State Manager\nprd.json (atomic)"]
        PROGRESSLOG["Progress Log Writer\nappend-only JSONL"]
        RESUMER["Resume Reconciler\nprd.json + progress.log"]
    end

    subgraph PROVIDERS["Provider Layer"]
        direction LR
        CLAUDE["claude CLI\nAgentCli + CompletionCli"]
        CODEX["codex CLI\nAgentCli + CompletionCli"]
    end

    subgraph ROLES["Roles (v1)"]
        direction LR
        EXECUTOR["executor\nAgentCli"]
        ANALYZER["analyzer\nCompletionCli\n--analyze + --analyze --fix"]
    end

    subgraph CHECKS["Verification Checks (parallel within a task)"]
        direction LR
        SHELL["shell\nexit 0"]
        HTTP["http\nstatus match"]
        FILECHECK["file-exists"]
        GREP["grep\nregex"]
        MANUAL["manual\nskipped (signal only)"]
    end

    subgraph STATE[".zurdo/<slug>/ — run state"]
        direction LR
        PRDJSON["prd.json\nterminal source of truth"]
        PROGLOG["progress.log\nappend-only JSONL"]
        LOCKFILE["lock\npid + start time"]
        ITERFILES["iterations/\n<task-id>-<attempt>.{out,err}"]
        REPORTS["reports/<ts>.{json,md}"]
    end

    subgraph ANALYZESTATE[".zurdo/<slug>/analyze-iterations/ — fix-loop audit trail"]
        direction LR
        ITERMD["<n>.md\nper-iteration PRD"]
        ITERFIND["<n>.findings.json\nanalyze output for iter <n-1>"]
        PARSEERR["<n>.parse-error.txt\nexit 8 only"]
        FIXSUMMARY["summary.json\ntrajectory · tokens · cost"]
    end

    PROPOSED["<prd>.proposed.md\nsibling file (next to PRD)"]

    subgraph OUTPUT["Output"]
        direction LR
        STDERR["stderr\nhuman log · thrash diagnostic"]
        STDOUT["stdout\nrun summary · fix-loop progress"]
        REPORTFILE["report file\nauto-written"]
    end

    PRD --> PARSER
    CONFIG --> CLI
    SKILLS --> SKILLINSTALLER
    CLI --> PARSER
    CLI --> LOCK
    LOCK --> LOCKFILE
    PARSER --> VALIDATOR
    VALIDATOR --> DEPGRAPH
    DEPGRAPH --> SCHEDULER
    SCHEDULER --> SKILLINSTALLER
    SKILLINSTALLER --> PROMPTBUILDER
    PROMPTBUILDER --> AGENT

    EXECUTOR --> AGENT

    SCHEDULER --> PREFLIGHT
    PREFLIGHT -->|all pass| STATEMGR
    PREFLIGHT -->|some fail| AGENT
    AGENT --> CLAUDE & CODEX
    CLAUDE & CODEX --> CLASSIFIER
    CLASSIFIER -->|Transient| AGENT
    CLASSIFIER -->|Permanent / Success| VERIFIER

    VERIFIER --> SHELL & HTTP & FILECHECK & GREP & MANUAL
    VERIFIER --> RETRY
    RETRY -->|criteria fail, budget left| AGENT
    RETRY -->|max attempts hit| STATEMGR
    RETRY -->|all pass| STATEMGR

    AGENT --> ITERFILES
    AGENT --> PROGRESSLOG
    VERIFIER --> PROGRESSLOG
    RETRY --> PROGRESSLOG
    STATEMGR --> PRDJSON
    PROGRESSLOG --> PROGLOG

    RESUMER --> PRDJSON
    RESUMER --> PROGLOG
    RESUMER --> ITERFILES
    RESUMER --> SCHEDULER
    ITERFILES --> PROMPTBUILDER

    STATEMGR --> STDOUT
    STATEMGR --> REPORTFILE
    REPORTFILE --> REPORTS
    CLI --> STDERR

    CLI -->|--analyze --fix| FIXLOOPBOX
    PRD --> FIXLOOPBOX
    ANALYZER --> FIXLOOPBOX
    FIXLOOPBOX --> CLAUDE & CODEX
    FIXLOOPBOX --> ITERMD
    FIXLOOPBOX --> ITERFIND
    FIXLOOPBOX --> PARSEERR
    FIXLOOPBOX --> FIXSUMMARY
    FIXLOOPBOX --> PROPOSED
    FIXLOOPBOX --> STDOUT
    FIXLOOPBOX --> STDERR
    FIXLOOPBOX -->|user `y` on TTY: mv| PRD
```

## Fix loop detail (`--analyze --fix`)

```mermaid
flowchart TD
    subgraph FIXINPUT["Input"]
        FIXPRD["PRD File\n.md (iteration n-1)"]
        FIXFINDINGS["Prior-iter findings\nerrors=0, warnings>0"]
    end

    subgraph FIXCORE["Fix Loop Orchestrator"]
        FIXORCH["Orchestrator\n--max-iterations N (default 5)\nthrash window = 3"]
        FIXPREFLIGHT["Fix Pre-flight\nerrors gate · clean gate"]
        ANALYZEPASS["Analyze Pass\n§9.2 deterministic + LLM"]
        FIXPROMPT["Fix Prompt Builder\nPRD verbatim\n+ findings + suggestions\n+ preservation block\n(IDs · order · Effort · Depends-on)"]
        FIXINVOKE["Fixer Invoker\nCompletionCli"]
        FIXPARSE["Response Parser\nstrict grammar §2.2"]
        FIXTRAJ["Warning Trajectory\n(task_id, category, criterion_idx)\n[NEW] · [PERSISTS] · [RESOLVED]"]
        FIXTERM["Termination Check\nclean=0 · cap=6 · thrash=7 · parse-fail=8"]
        FIXPROMOTE["Proposed File Writer\nwrite <prd>.proposed.md\n(overwrite prior silently)"]
        FIXAPPROVE["Approval Gate\nTTY: y/N prompt with diff\nnon-TTY or --no-prompt: skip"]
    end

    subgraph FIXANALYZER["Analyzer Role"]
        FIXANALYZERROLE["CompletionCli\nclaude · codex"]
    end

    subgraph FIXARTIFACTS[".zurdo/<slug>/analyze-iterations/"]
        direction LR
        A_ITERMD["<n>.md\nper-iteration PRD"]
        A_ITERFIND["<n>.findings.json"]
        A_PARSEERR["<n>.parse-error.txt\n(exit 8 only)"]
        A_SUMMARY["summary.json\nexit code · iterations\nper-warning trajectory\ntokens · cost"]
    end

    subgraph FIXOUT["Output"]
        F_PROPOSED["<prd>.proposed.md\nsibling next to PRD"]
        F_STDOUT["stdout\nper-iteration progress\n[NEW]·[PERSISTS]·[RESOLVED] tags"]
        F_STDERR["stderr\nthrash diagnostic (exit 7)\nparse-fail message (exit 8)"]
        F_PRDMV["<prd>.md\n(overwritten only on user `y`)"]
    end

    FIXPRD --> FIXORCH
    FIXORCH --> FIXPREFLIGHT
    FIXPREFLIGHT --> ANALYZEPASS
    ANALYZEPASS -->|errors > 0| F_STDERR
    ANALYZEPASS -->|warnings = 0 on iter 1| F_STDOUT
    ANALYZEPASS -->|warnings present| FIXPROMPT
    FIXFINDINGS --> FIXPROMPT

    FIXPROMPT --> FIXINVOKE
    FIXANALYZERROLE --> FIXINVOKE
    FIXINVOKE --> FIXPARSE
    FIXPARSE -->|ok| A_ITERMD
    FIXPARSE -->|ok| ANALYZEPASS
    FIXPARSE -->|unparseable| A_PARSEERR
    FIXPARSE -->|unparseable| FIXTERM

    ANALYZEPASS --> A_ITERFIND
    ANALYZEPASS --> FIXTRAJ
    FIXTRAJ --> F_STDOUT
    FIXTRAJ --> FIXTERM

    FIXTERM -->|continue| FIXPROMPT
    FIXTERM -->|halt: clean=0| FIXPROMOTE
    FIXTERM -->|halt: cap=6| FIXPROMOTE
    FIXTERM -->|halt: thrash=7| FIXPROMOTE
    FIXTERM -->|halt: thrash=7| F_STDERR
    FIXTERM -->|halt: parse-fail=8 → promote iter n-1| FIXPROMOTE
    FIXTERM -->|halt: parse-fail=8| F_STDERR
    FIXTERM --> A_SUMMARY

    FIXPROMOTE --> F_PROPOSED
    FIXPROMOTE --> FIXAPPROVE
    FIXAPPROVE -->|y on TTY: mv| F_PRDMV
    FIXAPPROVE -->|N · non-TTY · --no-prompt| F_STDOUT
```

### Invariants captured in the fix-loop view

- **`CompletionCli` only.** No `AgentCli`, no working-tree mutation. The fixer rewrites the PRD text in its response; Zurdo writes the result.
- **Decoupled from run state.** The fix loop never reads or writes `prd.json` / `progress.log` / `iterations/` / `reports/`. Its audit trail lives in `.zurdo/<slug>/analyze-iterations/` and is overwritten on each invocation.
- **PRD mutation is single-edged.** The only arrow that overwrites `<prd>.md` is `FIXAPPROVE -->|y on TTY: mv| <prd>.md`. Every other halt path writes to the sibling `<prd>.proposed.md`.
- **Preservation block enforced at parse.** Task IDs, declaration order, `Effort`, and `Depends-on` violations land in the parse-fail path (exit `8`), not silently in `summary.json`.
- **Trajectory is the bridge.** `(task_id, finding_category, criterion_idx)` is what makes `[PERSISTS]` recognize a rewritten criterion as the same warning across iterations — and what feeds the §9.7.6 thrash diagnostic.
