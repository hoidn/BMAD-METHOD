# Local Schema Registry

This repository uses a lightweight, in‑repo schema registry under `schemas/` for workflow‑level content contracts (e.g., QA verdicts). These schemas help agents produce structured outputs and enable deterministic validation in workflows.

## Conventions
- Location: `schemas/` at repo root.
- `$id` format: stable URNs, e.g., `urn:orchestrate:<name>:v<major>` (bump major on breaking changes).
- Versioning: keep multiple versions side‑by‑side (e.g., `qa_verdict.schema.json` → next becomes `qa_verdict.v2.schema.json` or new `$id`).
- Paths in outputs: when schemas reference file paths, they should be relative to WORKSPACE.

## Example: QA Verdict Schema
- File: `schemas/qa_verdict.schema.json`
- Shape (simplified):
  - `approved` (boolean, required)
  - `reason` (string)
  - `task_id` (string)
  - `issues[]` items: `{ message, severity: info|warn|error, path? }`
  - `outputs[]` items: strings (paths relative to WORKSPACE)

Minimal valid JSON:
```json
{
  "approved": true,
  "reason": "Unit + QA checks passed",
  "task_id": "demo_001",
  "issues": [],
  "outputs": ["artifacts/engineer/impl.py"]
}
```

## Using Schemas in Workflows

Two robust patterns are supported today via normal steps:

1) STDOUT JSON gate (fast path)
- Ask the agent to print only JSON to STDOUT, then validate via step assertions.
- Recommended step pattern:
```yaml
- name: QAReview
  provider: "claude"
  input_file: "prompts/qa/review.md"
  output_capture: json              # parse fails if non‑JSON
  depends_on:
    required: ["schemas/qa_verdict.schema.json"]
    inject:
      mode: content
      instruction: "Conform to this JSON Schema. Output ONLY JSON to STDOUT."
- name: AssertApproved
  command: ["bash","-lc","test \"${steps.QAReview.json.approved}\" = \"true\""]
```

2) Verdict file gate (robust path)
- Ask the agent to write JSON to a file; wait for it and assert via jq/validator.
```yaml
- name: WaitForVerdict
  wait_for:
    glob: "inbox/qa/results/${context.task_id}.json"
    timeout_sec: 3600
- name: AssertApproved
  command: ["bash","-lc","jq -e '.approved == true' inbox/qa/results/${context.task_id}.json >/dev/null"]
```

## Tooling Suggestions
- `jq` for simple assertions.
- `ajv` or `jsonschema` for full JSON Schema validation, if desired.
- Keep prompts strict: “Output only a single JSON object. No prose, no code fences.”

## Notes
- These schemas are application‑level conventions; the v1.1/1.1.1 DSL does not mandate them.
- A planned v1.3 DSL hook (output_schema/output_require) can make schema/requirements declarative on JSON steps.
