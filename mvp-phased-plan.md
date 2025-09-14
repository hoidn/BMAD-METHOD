# Multi-Agent Orchestrator — MVP Scope, Cuts, and Phased Plan

Document version: 1.0
Inputs considered: MULTI_AGENT_ORCHESTRATION_V1.1_SPEC.md (updated), arch.md, original story (Python control loop; agents hand off work via files under ./inbox/*)

## 1) MVP Focus and Non-Goals

- MVP goal (from story)
  - A Python control loop orchestrates 3 agents (architect, engineer, qa) that communicate by writing/reading files in per-agent directories.
  - Orchestrator calls headless LLM CLIs (e.g., `claude -p`, `gemini -p`) for each action.
  - Agents can block waiting for replies (engineer waits for qa feedback, both can ask architect, etc.).

- Minimal features required
  - YAML subset for linear flows (steps: provider, command for file writes, wait_for) with basic validation.
  - File protocol and directory structure for agent handoffs using `inbox/{agent}/`, `processed/{ts}/`, `failed/{ts}/`, and `.task` files.
  - Atomic task creation and movement to processed/failed.
  - Blocking `wait_for` for expected files with timeout (exclusive: cannot combine with command/provider/for_each on same step).
  - Headless provider invocation that passes prompt content as a single argv token (claude/gemini) or via stdin (codex) and captures stdout to a log file.
  - Path safety: reject absolute paths and any path containing `..`; follow symlinks but reject if resolved path escapes WORKSPACE.
  - POSIX glob semantics only (`*`, `?`); no recursive `**`.
  - Mutual exclusivity: steps use either `provider` or `command`, not both; `command_override` is disallowed (use `command`).
  - Simple, linear control flow sufficient to demo architect → engineer → qa loop with at least one round-trip (engineer → qa feedback → engineer).

- Non-goals for MVP (can be added later)
  - Full YAML DSL parity with v1.1 (branching/goto, for_each, pointer grammar, complex substitution, dependency injection content mode).
  - Full run state integrity suite with backups/repair/resume (resume optional later; backups behind flag).
  - Extensive observability (progress UI, trace IDs, metrics catalog, JSON output mode).
  - CLI extras like `watch`, `run-step`, `--json`, and `on-error interactive`.
  - Provider registry and configuration defaults; start with direct argv construction.


## 2) Potential Overengineering in arch.md for MVP

- Provider Registry and Templates
  - Overkill initially. MVP can directly assemble `claude`/`gemini` argv per step without a registry.

- Rich Variable Model and Pointer Grammar
  - Unnecessary for MVP. Start with minimal context replacement (e.g., `${run.timestamp_utc}`) or none. Hardcode simple flows.

- Dependency Validation and Injection
  - Useful but not required for a working demo. Defer both validation and prompt injection.

- Full State Manager with Backups/Repair/Resume
  - MVP can persist a simple run log; no backups/repair/resume needed initially.

- Output Capture Modes (lines/json) with Limits and Spillover
  - MVP can capture text only, with a single size cap and no spill files.

- Broad CLI Surface (`run`, `resume`, `run-step`, `watch`, `clean`, `--json`, `--trace`)
  - MVP needs only `run` with a few flags (`--debug`, `--timeout`), no `resume/watch/clean`.

- Advanced Error Handling and Retries
  - MVP: single attempt per step. Add retries later.

- Extensive Metrics and Progress UI
  - MVP: basic logging is sufficient.


## 3) MVP Directory and Protocol (aligned with spec)

- Directories
  - `inbox/architect/` — architect inbox (receives questions/tasks)
  - `inbox/engineer/` — engineer inbox
  - `inbox/qa/` — qa inbox
  - `artifacts/{architect,engineer,qa}/` — outputs created by provider calls
  - `processed/{timestamp}/` — completed task files
  - `failed/{timestamp}/` — failed/non-retryable task files

- Task files protocol
  - Writer: create `*.tmp` then atomic rename to `*.task` in recipient’s inbox.
  - Processor on success: move `*.task` to `processed/{ts}/` (keep full filename).
  - Processor on non-retryable failure: move to `failed/{ts}/`.
  - Content: freeform text; JSON recommended for structured messages later.

- Blocking wait
  - Orchestrator supports wait_for: poll for files matching a POSIX glob in a specific inbox with `timeout_sec` and `poll_ms`.


## 4) MVP Control Loop — Minimal YAML-driven Flow

1) Architect step: invoke provider with a design prompt → write outputs (e.g., system_design.md) to `artifacts/architect/`.
2) Orchestrator creates an engineer task in `inbox/engineer/` listing design outputs or instructions.
3) Orchestrator invokes engineer provider with a generic implement prompt; provider writes `src/impl.py` (or similar).
4) Orchestrator creates a QA review task in `inbox/qa/` referencing the engineer output.
5) Orchestrator waits for a QA feedback file to appear in `inbox/engineer/` or `inbox/qa/` (choose one; MVP: QA writes to `inbox/engineer/` as feedback task).
6) Orchestrator invokes engineer provider again to address feedback (optional for MVP to keep it simple), then completes.

MVP uses a small YAML subset to define a linear flow with `provider`, `command` (for writing tasks), and `wait_for`. A hardcoded run script may be used for development, but the primary demo is YAML-driven.


## 5) Implementation Order (dependency- and value-driven)

1) Minimal YAML Loader + Validation (must-have)
  - Parse a small YAML subset: steps of types `provider`, `command`, and `wait_for` only.
  - Validate path safety (no absolute paths or `..`; symlink resolution must stay inside WORKSPACE).
  - Enforce wait_for exclusivity (cannot combine with command/provider/for_each on same step).
  - Restrict glob semantics to POSIX `*` and `?` only.
  - Enforce provider/command mutual exclusivity; reject any usage of `command_override`.

2) File System Primitives (must-have)
  - Atomic write (tmp→rename), safe mkdir, move to processed/failed with timestamped folder.
  - Minimal glob/listing for inbox.

3) Wait/Poll Utility (must-have)
  - Poll for matching files with timeout/poll interval; return list of matches.

4) Provider Invocation Wrapper (must-have)
  - Function to call `claude`/`gemini` via argv token (`${PROMPT}`) or `codex` via stdin based on provider `input_mode`; capture stdout to a log file and return exit code.

5) Orchestrator Core (must-have)
  - Execute the linear YAML-defined flow described in Section 4 with robust logging and error checks.

6) Minimal CLI Entrypoint (nice-to-have for MVP)
  - `python -m orchestrate run` style or a simple script entry; flags: `--debug`, `--timeout`.

7) Optional Dev Scaffolding
  - Temporary hardcoded script for development parity with YAML-defined demo.


## 6) Phased Implementation Plan

### Phase 0 — Groundwork and Scaffolding
- Create Python package structure: `orchestrator/` with modules: `fsq.py` (queue ops), `wait.py`, `provider.py` (CLI calls), `core.py` (control loop), `log.py`.
- Add a simple entry script `orchestrate.py` invoking a demo run.
- Decide provider to target first (Claude recommended) and ensure CLI reachable.

Deliverable: demo script runs, logs to console.

### Phase 1 — Core MVP Flow (Story Demo)
- Implement FS primitives: write_task(tmp→rename), list_tasks(glob), move_processed/move_failed(ts dir).
- Implement `wait_for(glob, timeout_sec, poll_ms, min_count=1)` returning paths and wait metrics.
- Implement `call_provider(prompt_text: str, model: str, stdout_log_path: str)`; support `input_file` helper to read prompt from disk.
- Implement `core.run_demo()`:
  - Architect: call provider with design prompt; save outputs under `artifacts/architect/`.
  - Write engineer task file in `inbox/engineer/`.
  - Engineer: call provider with implement prompt; output `src/impl.py`.
  - Write QA review task in `inbox/qa/` referencing `src/impl.py`.
  - Wait for QA feedback task for engineer (`inbox/engineer/*.task`) with timeout.
  - If feedback arrives, log it; optionally make a second engineer call (optional in MVP).
- Logging: minimal timestamps, step names, exit codes.

Deliverable: YAML-driven end-to-end demonstration of file handoff and a blocking wait.

### Phase 2 — YAML Polish and Basic Context
- Add optional `providers` defaults (e.g., default model name) without a full registry.
- Add simple variable substitution for `${run.timestamp_utc}` and `${context.<k>}`.
- Keep linear flow only; no branching/for_each.

Deliverable: Cleaner YAML ergonomics with minimal defaults and context.

### Phase 3 — Basic State Tracking and Resume (Optional)
- Persist a minimal `state.json` under `.orchestrate/runs/<run_id>/` with step names, exit codes, timestamps.
- Atomic write (tmp→rename) but no backups/repair.
- Implement `resume <run_id>` to skip already completed steps in linear flows.

Deliverable: Can resume a partially completed run.

### Phase 4 — Dependency Validation (Required) and Simple Injection (Optional)
- Add `depends_on.required` validation for file existence/globs before provider steps.
- Optionally implement list-mode injection: prepend a simple instruction and a list of matched files to the prompt string.

Deliverable: Fast fail on missing inputs; optional improved prompt context.

### Phase 5 — Provider Configuration and Output Capture Modes
- Add provider defaults (e.g., default model) and per-step params.
- Support providers with `input_mode: 'argv'` (e.g., claude/gemini) and `input_mode: 'stdin'` (e.g., codex).
- Support output capture modes: text (default) and lines; JSON capture can wait until Phase 6.
- Add truncation limits; large outputs log to a sidecar file if needed.

Deliverable: More flexible provider usage (argv and stdin providers); better control of outputs.

### Phase 6 — Branching, for_each, and Pointers (Toward Spec)
- Implement `when.equals`, `on.success/failure goto` for branching; mark false conditions as `skipped`.
- Implement `for_each.items_from` with pointer grammar for `.lines` arrays only.
- Add `steps.<name>.output/lines` in the variable model.

Deliverable: Can execute simple non-linear flows and arrays of tasks.

### Phase 7 — Observability and Safety Rails
- Logging upgrades (progress view, debug traces, secret masking).
- CLI enhancements: retries and `--on-error stop|continue` (interactive later), `clean`, `archive-processed` with safeguards; `watch` and `--json` optional.
- State backups behind `--backup-state` and repair mode if Phase 3 resume is adopted.

Deliverable: Production-grade ergonomics and safety comparable to v1.1 spec.


## 7) Cut List Summary (Defer Until ≥ Phase 4)
- Provider registry/templates → ≥ Phase 5.
- Full variable model with precedence and escaping → ≥ Phase 6.
- Dependency injection (content mode, custom instructions) → ≥ Phase 5–6.
- JSON output capture and pointer into `.json.*` → ≥ Phase 6.
- Full state integrity suite (backups, repair) → ≥ Phase 7.
- Advanced CLI and observability (watch, trace, metrics) → ≥ Phase 7.


## 8) Risks and Mitigations
- Provider CLI availability and auth
  - Mitigation: Fail fast with clear error if CLI missing; document env vars.
- File protocol ambiguity
  - Mitigation: Codify `.task` extension, tmp→rename, and placement conventions in README for agents.
- Scope creep from spec parity
  - Mitigation: Enforce this phased plan; only add features by phase.


## 9) Quick Start Checklists (MVP)
- Folders: `inbox/{architect,engineer,qa}/`, `processed/`, `failed/`, `artifacts/`, `src/`
- Prompts: `prompts/architect/design.md`, `prompts/engineer/implement.md`, `prompts/qa/review.md`
- Configure provider CLI (e.g., `claude` model and auth)
- Run: `orchestrate run workflow.yaml` → produces design, engineer impl, QA task, waits for feedback
