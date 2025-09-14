# Orchestrator Q&A — Decisions, Changes, and Open Items

This document is the authoritative handoff for decisions made while implementing and evolving the orchestration spec and example workflows. It includes concrete references, examples, and next steps.

Sections:
- Decided + Implemented (with anchors, examples, and acceptance impact)
- Decided but Not Yet Integrated (actions, owners, targets)
- Addressed but Undecided (important open questions)
- Run Instructions for examples
- Schema Registry Conventions
- Risks & Edge Cases

## 1) Decided + Implemented (Spec / Prompts / Schemas / Examples)

Q: Reduce queue‑management boilerplate without hiding intent?
A: Yes — add an opt‑in, version‑gated lifecycle helper at the end of each `for_each` iteration.
- Decision: Declarative block `for_each.on_item_complete.success.move_to` and `.failure.move_to`.
- Rationale: Cut boilerplate `mkdir`/`mv` while preserving explicitness and transactional control.
- Spec changes: MULTI_AGENT_ORCHESTRATION_V1.1_SPEC.md
  - “Task Queue System” note (post‑MVP planned)
  - Step Schema: future v1.2 block under `for_each`
  - New section “Declarative Task Lifecycle for for_each (v1.2)”
  - Future Acceptance (v1.2)
- DSL version: Planned v1.2 (workflows must declare `version: "1.2"`).
- State: Per‑iteration `lifecycle` object (`result`, `action`, `from`, `to`, `action_applied`, `error?`).
- Migration: MVP stays manual; feature is opt‑in and version‑gated.
- Acceptance impact (planned): success/failure moves, goto escape semantics, path safety, idempotency, substitution.

Example (v1.2 planned):
```yaml
version: "1.2"
steps:
  - name: List
    command: ["find", "inbox/engineer", "-name", "*.task", "-type", "f"]
    output_capture: lines

  - name: Process
    for_each:
      items_from: "steps.List.lines"
      as: task_file
      on_item_complete:
        success: { move_to: "processed/${run.timestamp_utc}" }
        failure: { move_to: "failed/${run.timestamp_utc}" }
      steps:
        - name: Work
          command: ["bash", "-lc", "do_work '${task_file}'"]
        - name: Gate
          command: ["bash", "-lc", "test -f outputs/$(basename \"${task_file}\").ok"]
          on: { failure: { goto: _end } }
```

Resulting state fragment:
```json
{
  "steps": {
    "Process": [
      {
        "Work": {"status":"completed","exit_code":0},
        "Gate": {"status":"completed","exit_code":0},
        "lifecycle": {
          "result": "success",
          "action": "move",
          "from": "inbox/engineer/task_001.task",
          "to": "processed/20250115T143022Z/task_001.task",
          "action_applied": true
        }
      }
    ]
  }
}
```

Q: Make version gating explicit?
A: Yes — add a summary table and enforcement note.
- Decision: “Version Gating Summary” table listing 1.1, 1.1.1, 1.2 (planned), 1.3 (planned).
- Rationale: One place to audit availability and validation behavior.
- Spec changes: Table added under Workflow Schema section.

Q: Validate agent JSON outputs declaratively?
A: Yes — as a narrow, post‑capture hook on JSON steps.
- Decision: Planned v1.3 fields `output_schema` and `output_require`.
- Rationale: Remove boilerplate (ajv/jq steps) while keeping behavior local to the step.
- Spec changes: Step Schema future fields; “Future Acceptance (v1.3)”.
- DSL version: Planned v1.3; only with `output_capture: json` and `allow_parse_error: false`.

Example (v1.3 planned):
```yaml
- name: QAReview
  provider: "claude"
  input_file: "prompts/qa/review.md"
  output_capture: json
  output_schema: "schemas/qa_verdict.schema.json"
  output_require:
    - pointer: "/approved"
      equals: true
```

Failure context shape (example):
```json
{
  "error": {
    "message": "JSON output validation failed",
    "exit_code": 2,
    "context": {
      "json_schema_errors": ["/approved: expected boolean, got string"],
      "json_require_failed": {"pointer":"/approved","reason":"equals:true"}
    }
  }
}
```

Q: Robust QA verdicts with Claude Code (no MCP)?
A: Yes — prompts + injected schema + deterministic asserts.
- Decision: Add reusable prompt and schema; provide two gating patterns.
- Changes:
  - prompts/qa/review.md (JSON‑only STDOUT contract)
  - schemas/qa_verdict.schema.json (local registry stub)
  - workflows/examples/qa_gating_stdout.yaml (stdout JSON gate)
  - workflows/examples/qa_gating_verdict_file.yaml (verdict file gate)

Q: Ralph prompts for this repo?
A: Yes — build and plan prompts tailored to the spec.
- Changes: prompts/ralph_orchestrator_PROMPT.md, prompts/ralph_orchestrator_PLAN_PROMPT.md

## 2) Decided but Not Yet Integrated

Q: Unify state example to include `error` and a concise field reference?
A: Yes.
- Owner: TBD
- Target: v1.1.x docs refresh
- Acceptance: Top‑level state example includes `error` (message, exit_code, stdout_tail, stderr_tail, context), with a short “Step state fields” table.

Q: Centralize tunable limits (stdout text/lines caps, JSON buffer, injection cap) and expose config?
A: Yes.
- Owner: TBD
- Target: v1.1.x docs refresh
- Acceptance: Single “Limits” section listing sizes and how to tune via CLI flags or a `limits:` block; references throughout point to it.

Q: Lightweight schema registry hint in DSL (e.g., `schemas_dir`)?
A: Yes (optional, non‑normative hint).
- Owner: TBD
- Target: v1.2 docs
- Acceptance: Add `schemas_dir: "schemas"` to top‑level config with path‑safety note; non‑required.

Q: Add a “JSON Output Validation (v1.3)” example section with full YAML?
A: Yes.
- Owner: TBD
- Target: v1.3 docs
- Acceptance: Section shows a JSON step with `output_schema` + `output_require` and sample fail/pass.

Q: Runnable examples for v1.2 lifecycle helper?
A: Yes.
- Owner: TBD
- Target: v1.2 release branch
- Acceptance: Two examples (success/failure); resume idempotency verified; recorded `lifecycle` fields.

Q: Consolidate goto scoping + loop success semantics in one normative section?
A: Yes.
- Owner: TBD
- Target: v1.1.x docs refresh
- Acceptance: Section defines: success = all steps completed after retries with no escape goto; failure = any failed/timeout/escaped; recovery via `on.failure` counts as success if item completes.

## 3) Addressed but Undecided (Important Open Questions)

Q: Built‑in `complete_when` gate (e.g., wait on QA verdict before classifying success)?
A: Deferred. Explicit `wait_for` + assert remains the pattern; consider as future if demand is high.

Q: Extension namespace (`x-*`) to allow org‑local metadata without breaking strict validation?
A: Deferred. Pros: extensibility. Cons: weakens strictness. Might allow only under a dedicated metadata block.

Q: Cross‑platform glob/case‑sensitivity normalization?
A: Deferred. Keep “host filesystem governs” with a stronger note later.

Q: Built‑in path helpers (basename, dirname) to avoid shell‑specific idioms?
A: Deferred due to expression semantics creep. Continue using shell utils explicitly.

Q: Native MCP provider type vs. CLI wrapper?
A: Deferred. Wrapper recommended now; native provider requires design + version bump.

Q: Shared/remote schema registry?
A: Deferred. Local `schemas/` remains baseline; remote introduces portability/safety concerns.

---

## Run Instructions (Examples)

These assume `orchestrate` CLI is available in PATH and you are at repo root.

- STDOUT JSON gate example:
  - Command: `orchestrate run workflows/examples/qa_gating_stdout.yaml --debug`
  - Expected: Step `QAReviewStdout` succeeds with parsed JSON; `AssertQAApproved` gates success; `Report` echoes confirmation.

- Verdict file gate example:
  - Command: `orchestrate run workflows/examples/qa_gating_verdict_file.yaml --debug`
  - Expected: `QAReviewToFile` writes verdict JSON; `WaitForQAVerdict` unblocks; `ValidateSchemaBasics` and `AssertQAApproved` pass; `Report` echoes confirmation.

Notes:
- If your provider isn’t available, swap `provider: "claude"` with a local echo/mock or run with `--dry-run` to validate YAML.
- On parse/validation failures, see `.orchestrate/runs/<run_id>/state.json` and `logs/` for details.

## Schema Registry Conventions

- Location: `schemas/` at repo root. Example: `schemas/qa_verdict.schema.json`.
- `$id`: Stable, local URNs (e.g., `urn:orchestrate:qa_verdict:v1`).
- Injection: Use `depends_on.inject: { mode: content, instruction: "Conform to this schema. Output ONLY JSON to STDOUT." }` to supply schemas to providers.
- Validation patterns:
  - Fast path: `output_capture: json` → assert fields (today) or `output_schema` + `output_require` (v1.3 planned).
  - Robust path: agent writes verdict file; validate with `jq`/`ajv` in a command step.

## Risks & Edge Cases

- Goto scoping in loops: A `goto` that jumps outside the loop (or to `_end`) before finishing item steps classifies the item as failure (for v1.2 lifecycle). Recovery via `on.failure` that still reaches the end counts as success.
- strict_flow interplay: With `strict_flow: true` (default), a failed step halts unless `on.failure` supplies a `goto`. With `strict_flow: false` or `--on-error continue`, add explicit `when`/branching guards to avoid running postconditions erroneously.
- Idempotency/resume: Lifecycle actions (v1.2) must be idempotent; on resume, do not re‑move items already moved. Record `action_applied`.
- Path safety: All resolved paths (including `move_to`) must remain within WORKSPACE after following symlinks; reject absolute or parent‑escape paths.
- JSON capture limits: Very large JSONs (>1 MiB) fail parse; use file‑based verdicts for large payloads.

---

Changelog
- 2025‑09‑14: Expanded with examples, run instructions, registry conventions, action tracker, and risk notes.
