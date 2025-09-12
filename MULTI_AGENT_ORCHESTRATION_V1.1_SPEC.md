# Multi-Agent Workflow Orchestration Framework - v1.1 Specification

**Version:** 1.1  
**Status:** APPROVED FOR IMPLEMENTATION  
**Date:** January 2025

---

## Executive Summary

A developer-centric orchestrator for rapidly building, testing, and iterating on AI agent and CLI tool workflows. Version 1.1 enhances multi-agent coordination through filesystem-backed patterns while maintaining deterministic, sequential execution.

**Core Principles:**
- **Simplicity & Predictability**: Sequential execution with explicit paths
- **Zero Infrastructure**: Just Python + YAML + filesystem
- **Observable by Design**: Filesystem as message queue, structured status files
- **Safe by Default**: No shell interpretation, explicit secrets management

---

## Architecture Overview

### Directory Layout

```
workspace/
├── src/                    # User source code
├── prompts/               # Reusable prompt templates
├── artifacts/             # Agent-generated outputs
│   ├── architect/
│   ├── engineer/
│   └── qa/
├── inbox/                 # Agent work queues
│   ├── architect/
│   ├── engineer/
│   └── qa/
├── processed/             # Completed work items
│   └── {timestamp}/
└── failed/               # Failed work items (quarantine)
    └── {timestamp}/
```

**Path Resolution Rule:** All user-declared paths remain explicit and resolve against WORKSPACE. No auto-prefixing based on agent.

### Inbox Semantics

**Writing Tasks:**
1. Create as `*.tmp` file
2. Atomic rename to `*.task`

**Processing Results:**
- **Success:** `mv <task> processed/{timestamp}/`
- **Failure (non-retryable):** `mv <task> failed/{timestamp}/`

**File Content:** Freeform text; JSON recommended for structured data

**Configuration:**
```yaml
# Top-level workflow config (with defaults)
inbox_dir: "inbox"
processed_dir: "processed"
failed_dir: "failed"
task_extension: ".task"
```

---

## CLI Contract

```bash
# Run workflow from beginning (v1.0 behavior preserved)
orchestrate run workflows/demo.yaml \
  --context key=value \
  --context-file context.json \
  --clean-processed \           # NEW: Empty processed/ before run
  --archive-processed output.zip # NEW: Archive processed/ on success

# Resume failed/interrupted run (v1.0 behavior preserved)
orchestrate resume <run_id>

# Execute single step (v1.0 behavior preserved)
orchestrate run-step <step_name> --workflow workflows/demo.yaml

# Watch for changes and re-run (v1.0 behavior preserved)
orchestrate watch workflows/demo.yaml
```

**Safety:** The `--clean-processed` flag will only operate on the `WORKSPACE/processed/` directory and will refuse to run on any other path. The `--archive-processed` destination path must be outside of the `processed/` directory. If a destination is not provided, the archive defaults to `RUN_ROOT/processed.zip`.

---

## Variable Model (Simplified)

### Removed Namespace
- ~~`${env.*}`~~ - **REMOVED** for security and simplicity

### Retained Namespaces (precedence order)
1. **Run Scope**
   - `${run.timestamp_utc}` - The start time of the run, formatted as YYYYMMDDTHHMMSSZ

2. **Loop Scope**
   - `${item}` - Current iteration value
   - `${loop.index}` - Current iteration (0-based)
   - `${loop.total}` - Total iterations

3. **Step Results**
   - `${steps.<name>.exit_code}` - Step completion code
   - `${steps.<name>.output}` - Step stdout (text mode)
   - `${steps.<name>.lines}` - Array when `output_capture: lines`
   - `${steps.<name>.json}` - Object when `output_capture: json`
   - `${steps.<name>.duration}` - Execution time

4. **Context Variables**
   - `${context.<key>}` - Workflow-level variables

### Environment & Secrets

**Per-step environment injection (not substitution):**
```yaml
steps:
  - name: Build
    env:
      LOG_LEVEL: "debug"      # Non-secret env vars
    secrets:
      - GITHUB_TOKEN          # Secret env vars (masked in logs)
    command: ["npm", "run", "build"]
```

---

## Enhanced Step Schema

### New Fields

```yaml
# Agent label (optional, for documentation)
agent: "engineer"  # The agent field is an informational label for documentation and observability; it does not affect path resolution or any other execution behavior.

# Output capture mode
output_capture: "text"  # Default: text | lines | json

# Dynamic for-each from prior step
for_each:
  items_from: "steps.CheckInbox.lines"  # NEW: Reference array from prior step
  # OR traditional literal:
  items: ["a", "b", "c"]
  as: item
  steps: [...]

# Optional, only applies when output_capture: json
allow_parse_error: false

# Provider parameters (NEW)
provider: "claude"
provider_params:
  model: "claude-3-5-sonnet"
  max_tokens: 2048

# Command override (NEW)
command_override: ["claude", "-p", "Custom prompt"]

# Wait for files (NEW) - blocking primitive for inter-agent communication
wait_for:
  glob: "inbox/engineer/replies/*.task"  # File pattern to watch
  timeout_sec: 1800                      # Max wait time (default: 300)
  poll_ms: 500                          # Poll interval (default: 500)
  min_count: 1                          # Min files required (default: 1)
```

**Pointer Syntax:** The value must be a string in the format `steps.<StepName>.lines` or `steps.<StepName>.json[.<dot.path>]`. The referenced value must resolve to an array. Dot-paths do not support wildcards or advanced expressions in v1.1.

### Output Capture Modes

**`text` (default):** Traditional string capture
```json
{
  "output": "First line\nSecond line\n",
  "truncated": false
}
```

**`lines`:** Split on LF, array result
```json
{
  "output_capture": "lines",
  "lines": ["inbox/engineer/task1.task", "inbox/engineer/task2.task"],
  "truncated": false
}
```

**`json`:** Parse as JSON object
```json
{
  "output_capture": "json",
  "json": {"success": true, "files": ["a.py", "b.py"]},
  "truncated": false
}
```

Parse failure → exit code 2 unless `allow_parse_error: true`

**Limits:** To ensure stability, the following limits are enforced:
- **text:** The first 8 KB of stdout is stored in the state file. If the stream exceeds 1 MB, it is spilled to a log file and the `output` field is marked as truncated.
- **lines:** A maximum of 10,000 lines are stored. If exceeded, the `lines` array will contain the first 10,000 entries and a `truncated: true` flag will be set.
- **json:** The orchestrator will buffer up to 1 MB of stdout for parsing. If stdout exceeds this limit, parsing fails with exit code 2 (unless `allow_parse_error: true`). Invalid JSON also results in exit code 2.

**State Fields:** When `output_capture` is set to `lines` or `json`, the raw `output` field is omitted from the step's result in `state.json` to avoid data duplication.

---

## Provider Configuration

### Direct CLI Integration

**Workflow-level templates:**
```yaml
providers:
  claude:
    command: ["claude", "--model", "${model}", "--max-tokens", "${max_tokens}"]
    defaults:
      model: "claude-3-5-sonnet"
      max_tokens: 4096
  
  gemini:
    command: ["gemini", "--model", "${model}"]
    defaults:
      model: "gemini-pro"
```

**Step-level usage:**
```yaml
steps:
  - name: Analyze
    provider: "claude"
    provider_params:           # Override defaults
      model: "claude-3-5-haiku"
      max_tokens: 2048
    input_file: "prompts/analyze.md"
    output_file: "artifacts/architect/analysis.md"

  - name: CustomProvider
    command_override: ["claude", "-p", "Special prompt", "--model", "claude-3-5-sonnet"]
```

**Exit code mapping (unchanged from v1.0):**
- 0 = Success
- 1 = Retryable API error
- 2 = Invalid input (non-retryable)
- 124 = Timeout (retryable)

---

## Status Communication

### Status JSON Schema

```json
{
  "schema": "status/v1",
  "correlation_id": "uuid-or-opaque",
  "agent": "engineer",
  "run_id": "uuid",
  "step": "ImplementFeature",
  "timestamp": "2025-01-15T10:30:00Z",
  "success": true,
  "exit_code": 0,
  "outputs": ["artifacts/engineer/implementation.py"],
  "metrics": {
    "files_modified": 3,
    "lines_added": 150
  },
  "next_actions": [
    {
      "agent": "qa",
      "action": "review",
      "file": "inbox/qa/review_implementation.task",
      "priority": "high"
    }
  ],
  "message": "Feature implemented successfully"
}
```

**Default path:** `artifacts/<agent>/status.json` or `status_<step>.json`

**Path Convention:** All file paths included within a status JSON file (e.g., in the `outputs` array) must be relative to the WORKSPACE directory.

---

## Example: Multi-Agent Inbox Processing

```yaml
version: "1.1"
name: "multi_agent_feature_dev"
strict_flow: true

providers:
  claude:
    command: ["claude", "--model", "${model}", "--max-tokens", "${max_tokens}"]
    defaults:
      model: "claude-3-5-sonnet"
      max_tokens: 4096

steps:
  # Check for pending engineer tasks
  - name: CheckEngineerInbox
    command: ["find", "inbox/engineer", "-name", "*.task", "-type", "f"]
    output_capture: "lines"
    on:
      success:
        goto: ProcessEngineerTasks
      failure:
        goto: NoTasks

  # Process each task
  - name: ProcessEngineerTasks
    for_each:
      items_from: "steps.CheckEngineerInbox.lines"
      as: task_file
      steps:
        - name: ReadTask
          command: ["cat", "${task_file}"]
          output_file: "current_task.txt"
          
        - name: ImplementWithClaude
          agent: "engineer"
          provider: "claude"
          input_file: "artifacts/engineer/current_task.txt"
          output_file: "artifacts/engineer/impl_${loop.index}.py"
          
        - name: WriteStatus
          command: ["echo", '{"success": true, "task": "${task_file}"}']
          output_file: "artifacts/engineer/status_${loop.index}.json"
          output_capture: "json"
          
        - name: MoveToProcessed
          command: ["mv", "${task_file}", "processed/${run.timestamp_utc}_${loop.index}/"]
          
        - name: CreateQATask
          when:
            equals:
              left: "${steps.WriteStatus.json.success}"
              right: "true"
          command: ["echo", "Review impl_${loop.index}.py"]
          output_file: "inbox/qa/review_${loop.index}.task"

  - name: NoTasks
    command: ["echo", "No pending tasks"]
    on:
      success:
        goto: _end
```

---

## Migration from v1.0

### Breaking Changes
- `${env.*}` namespace removed → Use `${context.*}` or per-step `env:`

### New Features (backward compatible)
- `output_capture: lines|json` for structured data
- `items_from:` for dynamic iteration
- Provider templates and `command_override`
- Status JSON files
- Inbox/processed/failed directories
- `wait_for:` blocking file wait primitive

### Migration Steps
1. Replace `${env.VAR}` with `${context.VAR}` or add to step `env:`
2. Optionally reorganize artifacts under agent subdirectories
3. Incrementally adopt inbox pattern and status files

---

## v1.1 Acceptance Tests

1. **Lines capture:** `output_capture: lines` → `steps.X.lines[]` populated
2. **JSON capture:** `output_capture: json` → `steps.X.json` object available
3. **Dynamic for-each:** `items_from: "steps.List.lines"` iterates correctly
4. **Status schema:** Write/read status.json with v1 schema
5. **Inbox atomicity:** `*.tmp` → `rename()` → visible as `*.task`
6. **Processed/failed:** Success → `processed/{ts}/`, failure → `failed/{ts}/`
7. **No env namespace:** `${env.*}` rejected by schema validator
8. **Provider templates:** Template + defaults + params compose argv correctly
9. **Command override:** Replaces template entirely
10. **Clean processed:** `--clean-processed` empties directory
11. **Archive processed:** `--archive-processed` creates zip on success
12. **Pointer Grammar:** A workflow with `items_from: "steps.X.json.files"` correctly iterates over the nested `files` array
13. **JSON Oversize:** A step producing >1 MB of JSON correctly fails with exit code 2
14. **JSON Parse Error Flag:** The same step from above succeeds if `allow_parse_error: true` is set
15. **CLI Safety:** `orchestrate run --clean-processed` fails if the processed directory is configured outside WORKSPACE
16. **Wait for files:** `wait_for` step blocks until matching files appear or timeout
17. **Wait timeout:** `wait_for` with no matching files exits with code 124 after timeout
18. **Wait state tracking:** `wait_for` records `files`, `wait_duration`, `poll_count` in state.json

---

## Deferred to v2

### Concurrency
- Sequential execution only in v1.1
- v2 design note: Atomic `mv` as distributed lock for concurrent inbox processing

### Advanced Features
- While loops
- Parallel execution blocks
- Complex expression evaluation
- Event-driven triggers

---

## Implementation Order

1. **Schema Updates** (Day 1)
   - Remove env namespace validation
   - Add output_capture, items_from, agent fields
   - Add provider configuration schema

2. **Core Engine** (Day 2-3)
   - Output capture modes implementation
   - Dynamic for-each from arrays
   - Status file writing/reading

3. **Provider Integration** (Day 4)
   - Template system
   - Direct CLI execution
   - Command override support

4. **Inbox Pattern** (Day 5)
   - Atomic write helpers
   - Move to processed/failed
   - Directory initialization

5. **CLI Operations** (Day 6)
   - --clean-processed flag
   - --archive-processed flag
   - Directory structure validation

6. **Testing & Documentation** (Day 7)
   - Acceptance test suite
   - Migration guide
   - Example workflows

---

## Success Metrics

- All v1.0 workflows continue to run unchanged
- Inbox processing handles 100+ tasks sequentially
- Status files provide full execution visibility
- Zero auto-magic path mutations
- < 200 additional lines of code for v1.1 features

---

**END OF SPECIFICATION v1.1**