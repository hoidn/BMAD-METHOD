# Multi-Agent Workflow Orchestration Framework - Final Specification

# TODO how well does this spec doc represent intent of the initial <q>?
<q>
---

ultrathink how we could design an mvp workflow orchestration framwork with the following features:
multiple agents that can communicate and hand off work to each other by writing files. for example we might have 3 agents: architect, engineer and qa. each agent would have an inbox type subdir, something like this:
./agents/architect/
./agents/engineer/
./agents/qa/

they would interact through a protocol where each agent expects to recieve certain file inputs from the others. for example, engineer will request feedback from qa (and should block until the feedback arrives). qa and engineer can ask questions  to architect, etc. instead of trying to manage inter agent comm from within agent sessions there should be a pytohn control loop that orchestrates things and calls the agent in headless mode (claude -p or gemini -p) every time it wants to do something. 

a lot of the bmad yaml workflow config philosophy should be reusable but obviously the architecture will have to be different.

ultrathink about what i'm describing. does it make sense? do you have any questions?
</q>

## Executive Summary

A developer-centric orchestrator for rapidly building, testing, and iterating on AI agent and CLI tool workflows. The framework prioritizes simplicity and predictability through sequential execution, robust recovery mechanisms, and direct workspace integration, allowing developers to work with complex chains of tools as easily as they would with a simple script.

**Core Principles:**
- **Simplicity & Predictability**: Fresh runs execute steps sequentially, gated only by explicit when conditions. No caching is performed. The resume command uses the prior run log to skip steps that already succeeded and continue from the failure point.
- **Robust Recovery**: The primary mechanism for saving time is not caching, but the ability to reliably resume a failed workflow from the exact point of failure.
- **Workspace-Centric**: The tool operates directly in the developer's project directory, making inputs and outputs easy to find and manage.
- **Safe by Default**: Prevents common security issues without imposing restrictive limitations.
- **Zero Infrastructure**: Just Python + YAML. No databases, queues, or servers.

## MVP Scope (Ship First)

**Target**: Linux-only, Python 3.11+, PyYAML + jsonschema dependencies only

# TODO see <1>
**Goal**: Run 3-step workflows mixing CLI and LLM providers with conditional branching and sequential for-each loops.

### CLI Contract

```bash
# Run a workflow from the beginning. Executes every step.
orchestrate run workflows/demo.yaml \
  --context key=value \
  --context-file context.json

# Resume a failed or interrupted run from the point of failure.
orchestrate resume <run_id>

# Execute a single step from the workflow using a fresh ephemeral run_id. It loads the workflow and context (including set_context-independent defaults), executes only the named step, writes artifacts under WORKSPACE/artifacts/{StepName}/, and stores ephemeral logs under RUN_ROOT. It does not modify any existing state.json for prior runs.
orchestrate run-step <step_name>

# Watch for file changes and trigger a full re-run of the workflow.
orchestrate watch workflows/demo.yaml
```

## Developer Experience

**Automatic `.gitignore`:**
On the first run in a new project, the orchestrator will prompt to create or update the `.gitignore` file to include the `.orchestrator/` directory.

**Clear and Actionable Logging:**
Log output is designed for developers. When a step starts, it will be logged clearly:
`INFO: Step 'Analyze' starting.`
When a step completes, the outcome will be clear:
`INFO: Step 'Analyze' completed successfully in 45.2s.`

**Interactive Workflow:** 
- `watch` mode triggers full workflow re-runs on file changes
- `resume` command continues from point of failure
- `run-step` allows isolated step execution

# TODO does this lend itself to iterative improvement? e.g. maybe the user wants to create new items in a todo file or dir. how does this get propagated in a sensible way? there should be option for either an already-running agent to pick it up or to kick off an incremental work workflow
Watch (MVP): Implemented via polling (no extra deps). Every 500 ms, hash (mtime,size) across workflows/**/*.yaml and workspace/**/* (excluding .orchestrator/). On change, perform a fresh run.

### MVP Process Model

**Working Directory:**
- Default CWD: `workspace/` for all steps
- Artifacts are written to `workspace/artifacts/`

**Environment:**
Namespace precedence in substitution: loop → steps → context → env. Names never shadow across namespaces; each must be referenced explicitly (${context.FOO} vs ${env.FOO}).
Process environment: Only secrets explicitly allowlisted at workflow/step scope are exported to the child process. If an env var name collides with a secret name, the secret's value is used for the child process. Substitution precedence remains as defined above.

## Architecture

### Centralized Control Model

A single WorkflowRunner orchestrates all execution:
1. Loads and validates workflow definition against schema
2. Creates unique run isolation directory
3. Executes steps according to control flow rules
4. Persists state atomically between steps
5. Handles all branching and error recovery. (Parallelism is deferred to Phase 3.)

### Directory Layout

```
project/
├── .orchestrator/         # Internal state, logs, and run data
│   └── runs/
│       └── {run_id}/
│           ├── state.json # Run state and step results
│           └── logs/      # Structured logs
├── workflows/              # Workflow definitions
│   └── *.yaml             # Individual workflows
└── workspace/             # Main working directory for all steps
    ├── src/
    ├── prompts/          # Reusable prompt templates  
    └── artifacts/        # (Example) Output directory for generated files
```

## Path & Directory Policy (Canonical)

```
BASE = PROJECT_ROOT
WORKSPACE = BASE/workspace            # Step CWD for all executions
RUN_ROOT = BASE/.orchestrator/runs/{run_id}
```

Resolution rules:
- All user-declared relative paths (input_file, output_file, file_exists, prompt_file) resolve against WORKSPACE.
- Absolute paths and any path that resolves outside BASE are rejected with error code 3 (Path security violation).
- State, logs, and run artifacts for bookkeeping reside under RUN_ROOT.
- Artifacts produced for the project live under WORKSPACE/artifacts/{StepName}/ (not in RUN_ROOT).

**Directory Layout Rules:**
- **Default CWD**: All steps now execute with their Current Working Directory (cwd) set to workspace/
- **Artifacts**: Steps write artifacts directly into the workspace/ directory (e.g., workspace/artifacts/, workspace/dist/). All artifacts use workspace-relative paths
- **State Isolation**: All internal state, logs, and run-specific data are stored in the .orchestrator/ directory, which should be added to the project's .gitignore

## Syntax and Evaluation Model

### Variable Substitution

All `${...}` patterns are replaced with values from the defined namespaces. The evaluation process is designed for safety and predictability.

**Namespaces** (in precedence order):

1. **Loop Scope** (highest precedence)
   - `${item}` - Current iteration value in for_each
   - `${loop.index}` - Current iteration number (0-based)
   - `${loop.total}` - Total iterations
   - `${loop.iteration}` - While loop counter
   - `${loop.elapsed}` - Seconds since loop start

2. **Step Results**
   - `${steps.<name>.exit_code}` - Step completion code
   - `${steps.<name>.output}` - Step stdout (truncated)
   - `${steps.<name>.duration}` - Execution time in seconds

3. **Context Variables**
   - `${context.<key>}` - Workflow-level variables
   - Set at workflow start or via set_context steps

# TODO why do env variables need to be tracked?
4. **Environment Variables**
   - `${env.<name>}` - System environment variables
   - Must be explicitly allowlisted in workflow

**Evaluation Phases:**

1. **Parse & Dependency Analysis**: The orchestrator first parses all string fields to identify `${...}` references without resolving them. This builds a dependency graph for the step.

2. **Strict Resolution**: Before execution, the orchestrator resolves all dependencies.

**Missing Variable Policy:**

By default, a reference to a missing or undefined variable will cause the workflow to fail with an error (`E_VAR_MISSING`). This prevents silent failures from typos or logic errors.

This behavior can be overridden on a per-field basis by opting into a fallback mechanism:

```yaml
# Example of allowing a missing variable
command: ["echo", "User: ${context.user}", "--optional-flag=${context.flag}"]
allow_missing_vars:
  - context.flag  # This specific variable will resolve to "" if not set
```

**Escaping Rules:**
- `$$` - Resolves to a literal single dollar sign (`$`)
- `${{...}}` - The expression is deferred. The literal `${{...}}` is passed through, intended for downstream tools that have their own templating engines (e.g., Docker)
- Backslash escapes (`\`) are not processed by the substitution engine

**Substitution Scope:**
Variable substitution is applied recursively to all user-provided **string values** within the workflow definition. This includes strings in lists (e.g., `command` arguments) and strings as values in objects.

Substitution applies to string values including elements of command: [ ... ]. We do not spawn a shell; commands are executed as argv arrays. This forbids shell interpolation while still allowing safe ${...} substitution in arguments.

The following top-level workflow and step keys are exempt from substitution: `version`, `name`, `strict_flow`.

### Condition Model

**MVP Predicates Only** (all string values undergo `${...}` substitution first):

```yaml
# Check if step succeeded (exit code 0)
step_ok: "build"

# Check if file exists (relative to WORKSPACE)
file_exists: "artifacts/StepName/output.txt"

# String equality comparison
equals:
  left: "${context.branch}"
  right: "main"
```

file_exists resolves against WORKSPACE only (see Path & Directory Policy).

**Boolean Combinators:**

```yaml
when:
  all:                          # AND operation
    - step_ok: build
    - file_exists: artifacts/Build/app.js
  any:                          # OR operation
    - step_ok: test1
    - step_ok: test2
  not:                          # NOT operation
    file_exists: .halt
```

**Deferred to Phase 2:**
- `exit_code` comparisons
- `env_set` checks
- `contains` string matching
- `regex` pattern matching
- `number` numeric comparisons

**MVP I/O Contract:**

## Command I/O Semantics (MVP)

If input_file is set, the orchestrator opens WORKSPACE/<input_file> (text, UTF-8, errors='replace') and pipes it to child stdin. If not set, child stdin is closed.

If output_file is set, the orchestrator writes captured stdout to WORKSPACE/artifacts/{StepName}/<output_file> (parents auto-created, overwrite). stderr is captured separately under RUN_ROOT/logs/{StepName}-stderr.log.

provider steps follow the same input_file→stdin and output_file→artifact rules.

**Stdout/stderr capture:**
In-memory cap: 1MB per stream. On overflow, switch to streaming into RUN_ROOT/logs/{StepName}-stdout.log / ...-stderr.log. In state.json, store the first 8KB followed by "\n[truncated]" and record spill_stdout_path / spill_stderr_path with absolute paths under RUN_ROOT.

**Output file handling:**
- Parent directories auto-created
- Overwrites existing files
- Text mode only for MVP
- Written to `workspace/artifacts/{step_name}/{filename}`

### Control Flow

**MVP: Strict Mode Only** (`strict_flow: true` always): Every step must have explicit transitions

# TODO how is success / failure communicate by the agent?
**Transition Actions:**
```yaml
on:
  success:
    goto: NextStep      # Jump to named step
  failure:
    error: "Message"    # Terminate with error (exit 1)
  timeout:
    end: true          # Terminate successfully (exit 0)
```

**Special Targets:**
- `_start` - Explicit entry point
- `_end` - Normal termination
- `_error` - Error handler
- `_loop_break` - Exit current loop
- `_loop_continue` - Skip to next iteration

## Loop Specifications

### For-Each Loops (MVP: Sequential Only)

```yaml
for_each:
  items: ["file1", "file2", "file3"]  # Literal array only for MVP
  as: item                             # Iterator variable name
  steps:                               # Loop body
    - name: Process
      command: ["process", "${item}"]
      on:
        success:
          goto: _loop_continue
        failure:
          goto: _loop_break
```

**MVP Limitations:**
- Sequential execution only (no `parallel` option)
- Literal arrays only (no `${context.list}` references)
- Results stored in `steps.<name>.iterations[]`
- Special targets: `_loop_continue`, `_loop_break`

On completion, per-iteration results are persisted to steps.<loop_name>.iterations[] using the schema above.

### While Loops

**Deferred to Phase 2** - Not supported in MVP

## Parallel Execution

**Deferred to Phase 3** - MVP executes all steps sequentially only

## File and Path Resolution

### Path Types

# TODO how are agent / artifact files distinguished from source files?
1. **Workspace Files**: Project source files. Resolved relative to WORKSPACE.
   - Used for: source code, configs, static resources

2. **Artifact Files**: Generated files
   - Resolved relative to `workspace/artifacts/`
   - Used for: step outputs, intermediate results

3. **Prompt Files**: Reusable prompt templates
   - MVP Prompt Files: prompt_file points to a single file under WORKSPACE/prompts/. The file's bytes are piped to stdin. No includes or templating in MVP.

### Resolution Rules

```yaml
# Workspace file
input_file: src/main.py

# Artifact (previous step output, workspace-relative)
input_file: workspace/artifacts/build/output.txt

# Prompt template
prompt_file: prompts/analyze.md

# Absolute paths forbidden for security
input_file: /etc/passwd  # ERROR
```

## Provider Interface

To ensure a stable and controllable contract, the orchestrator will not call third-party CLIs directly. Instead, it will invoke standardized shims that are expected to be in the user's `PATH`.

**Shim Contract:**
- **Invocation:** `[provider-name]-shim --model [model_name] --max-tokens [number]`
- **Input (Prompt):** Passed via `stdin`
- **Output (Completion):** Written to `stdout`
- **Environment:** The shim will receive required secrets (e.g., `CLAUDE_API_KEY`) in its environment

**Exit code mapping (canonical):**
- 0  = Success
- 1  = Retryable API error (e.g., 5xx/network)
- 2  = Invalid input (non-retryable)
- 124 = Timeout (retryable if attempts > 1)
- All other codes map to "Execution error" (non-retryable by default in MVP)

# TODO <1> actually providers will be claude code and gemini cli and (optionally) openai codex
**MVP Example (`claude` provider):**
The orchestrator will construct and execute the following command:
```bash
claude-shim --model claude-3-haiku-20240307 --max-tokens 4000
```

The orchestrator is responsible for managing stdin/stdout and interpreting the exit code. The implementation of `claude-shim` is outside the scope of the orchestrator itself but is a required dependency for using the provider.

## State Management

### Run Isolation

Each workflow execution has a unique run ID (UUID v4) with:
- Isolated directory: `.orchestrator/runs/{run_id}/`
- Separate state file: `.orchestrator/runs/{run_id}/state.json`
- Shared artifacts: `workspace/artifacts/`
- Structured logs: `.orchestrator/runs/{run_id}/logs/`

### State Persistence

Atomic write: Serialize to state.json.tmp, fsync() the file, then rename() to state.json, then fsync() the parent directory. On startup, if a .tmp exists, discard it. If state.json is corrupt or missing required fields, abort with configuration error (exit 2).

**State Schema (MVP):**
```json
{
  "run_id": "uuid",
  "workflow_name": "demo",
  "status": "running|completed|failed",
  "started_at": "ISO-8601-UTC",
  "current_step": "StepName",
  "context": {},
  "steps": {
    "Process": {
      "status": "completed|failed",
      "exit_code": 0,
      "output": "truncated stdout",
      "duration": 45.2,
      "iterations": [
        {
          "index": 0,
          "item": "file1",
          "status": "completed|failed",
          "exit_code": 0,
          "duration": 1.23,
          "output": "truncated stdout"
        }
      ]
    }
  }
}
```

### MVP: No Locking

- Single process execution only
- No concurrent runs of same workflow
- No lock files needed

## State Management: Run Log & Recovery

The `state.json` file serves a single, critical purpose: it is a **run log** that enables failure recovery.

**State File (`state.json`):**
- Before each step runs, the orchestrator updates `state.json` to set the `current_step`.
- After each step completes, the orchestrator records the step's results (exit code, outputs, duration) in the state file.
- This provides a complete, step-by-step history of the run.

**The `resume` Command:**
The `resume` command is the primary way to handle failures and avoid re-running completed work.
1. The orchestrator loads the `state.json` from the specified `<run_id>`.
2. It identifies all steps that have already completed successfully. These steps are not re-executed.
3. It begins execution from the `current_step` that was marked at the time of failure.
4. The original context from the initial run is used.

## Error Handling

### Exit Codes

**MVP Exit Codes:**
- `0`: Successful completion
- `1`: Execution error
- `2`: Configuration error
- `3`: Path security violation
- `124`: Timeout

### Timeout Handling

Step Timeout (MVP): Default 300s per step. On timeout: send SIGTERM, wait 10s, then SIGKILL. Report exit code 124. No global workflow timeout. No cooperative cancellation tokens in MVP.

### Error Recovery

**MVP Retry Logic:**
```yaml
retry:
  attempts: 3  # Only field supported in MVP
```

Retry applies to exit codes {1,124} only, with fixed 2s backoff.

### Logging

**Structured JSON Logs:**

The log format includes event_seq and attempt_id to support deterministic reconstruction and future parallel runs., two fields are added:
- `event_seq`: A monotonic integer, assigned by a single atomic counter within the runner at the time of emission
- `attempt_id`: An integer indicating the retry attempt number for the step

```json
{
  "timestamp": "2024-01-01T12:00:00Z",
  "run_id": "550e8400-e29b-41d4-a716",
  "event_seq": 15,
  "level": "INFO",
  "step": "Build",
  "attempt_id": 1,
  "event": "step_complete",
  "duration": 45.2,
  "exit_code": 0
}
```

**Log Levels:**
- `ERROR`: Step failures, exceptions
- `WARNING`: Retries, timeouts, error conditions
- `INFO`: Step start/complete, transitions
- `DEBUG`: Variable substitution, evaluation details

## Security Model

### Command Execution

- Commands specified as arrays (no shell interpretation)
- No shell interpolation. ${...} substitution in command[] arguments is allowed and resolved by the orchestrator before exec.
- Temporary files for large inputs
- Output size limits enforced

### Secret Management

```yaml
secrets:                    # Workflow-level declaration
  - GITHUB_TOKEN
  - API_KEY

steps:
  - name: Deploy
    secrets:               # Step-specific allowlist
      - GITHUB_TOKEN
    command: ["deploy.sh"]
```

**Secret Handling:**
- Only explicitly listed secrets are available
- Secrets masked in logs (replaced with `***`)
- Never written to state file
- Passed via environment variables only

### Resource Limits (Deferred)

Beyond per-step timeout, resource limits are not enforced in MVP. Supplying a limits: block is a validation error.

### Filesystem & Path Safety

- **Workspace is Writable**: The workspace/ directory is fully writable by default. The concept of a read-only workspace is removed from the MVP
- **Path Resolution**: Paths in input_file or output_file are now resolved relative to the workspace/ directory
- **No Escape**: The orchestrator still forbids paths that resolve outside the project root (e.g., via ../ or absolute paths) to maintain project encapsulation
- **Symlink Policy**: Symlinks within workspace/ and prompts/ are disallowed in MVP. Configuration to enable them is deferred.

## MVP Workflow Schema

### Workflow Definition (MVP)

```yaml
# Required fields
version: "1.0"              # Schema version
name: "My Workflow"         # Workflow name
strict_flow: true           # Always true for MVP
steps: []                   # Step definitions

# Optional fields
context:                    # Initial variables
  key: value
secrets:                    # Required env vars
  - API_KEY
  - GITHUB_TOKEN
```

### Step Definition (MVP)

```yaml
name: "StepName"           # Required: unique identifier

# Execution (exactly one required)
command: ["cmd", "arg"]    # Array format only
# OR
provider: "claude"         # claude, openai, or gemini
# OR  
for_each:                  # Sequential loop
  items: ["a", "b", "c"]
  as: item
  steps: [...]
# OR
set_context:               # Update context variables
  key: "value"
  another_key: "${steps.prev.output}"

# Input/Output (optional)
input_file: "file.txt"     # Single input
output_file: "out.txt"     # Output destination

# Control Flow
when:                      # Optional condition
  step_ok: "PrevStep"
  
on:                        # Required transitions
  success:
    goto: "NextStep"
  failure:
    error: "message"

# Limits (optional)
timeout: 300               # Seconds (default: 300)
retry:
  attempts: 3              # Default: 1
allow_missing_vars:        # Optional list of vars that can be missing
  - context.optional_flag
```

**Note on `set_context`**: The `set_context` step performs a shallow merge into the global context. Keys are overwritten if they already exist. This is the only mechanism for dynamically modifying context during a run.

**Not in MVP:**
- `parallel` blocks
- `while` loops
- `input_files` (multiple)
- `limits` (beyond timeout)
- `catch` blocks
- Complex conditions

### MVP Condition Schema

For the MVP, the `when` block is validated against a stricter schema that only permits `step_ok`, `file_exists`, `equals`, and the boolean combinators `all`, `any`, `not`.

```json
{
  "definitions": {
    "mvp_condition": {
      "oneOf": [
        {"$ref": "#/definitions/all_op"},
        {"$ref": "#/definitions/any_op"},
        {"$ref": "#/definitions/not_op"},
        {"$ref": "#/definitions/step_ok_op"},
        {"$ref": "#/definitions/file_exists_op"},
        {"$ref": "#/definitions/equals_op"}
      ]
    },
    "all_op": { "type": "object", "properties": { "all": { "type": "array", "items": {"$ref": "#/definitions/mvp_condition"} } }, "required": ["all"] },
    "any_op": { "type": "object", "properties": { "any": { "type": "array", "items": {"$ref": "#/definitions/mvp_condition"} } }, "required": ["any"] },
    "not_op": { "type": "object", "properties": { "not": {"$ref": "#/definitions/mvp_condition"} }, "required": ["not"] },
    "step_ok_op": { "type": "object", "properties": { "step_ok": {"type": "string"} }, "required": ["step_ok"] },
    "file_exists_op": { "type": "object", "properties": { "file_exists": {"type": "string"} }, "required": ["file_exists"] },
    "equals_op": { "type": "object", "properties": { "equals": { "type": "object", "required": ["left", "right"], "properties": { "left": {"type": "string"}, "right": {"type": "string"} } } }, "required": ["equals"] }
  }
}
```

### Full Condition Schema (Reference for Phase 2+)

Conditions are structured predicates that can be composed with boolean operators:

```json
{
  "definitions": {
    "condition": {
      "oneOf": [
        {
          "type": "object",
          "properties": {
            "all": {
              "type": "array",
              "items": {"$ref": "#/definitions/condition"}
            }
          },
          "required": ["all"]
        },
        {
          "type": "object",
          "properties": {
            "any": {
              "type": "array",
              "items": {"$ref": "#/definitions/condition"}
            }
          },
          "required": ["any"]
        },
        {
          "type": "object",
          "properties": {
            "not": {"$ref": "#/definitions/condition"}
          },
          "required": ["not"]
        },
        {
          "type": "object",
          "properties": {
            "step_ok": {"type": "string"}
          },
          "required": ["step_ok"]
        },
        {
          "type": "object",
          "properties": {
            "exit_code": {
              "type": "object",
              "required": ["step", "op", "value"],
              "properties": {
                "step": {"type": "string"},
                "op": {"enum": ["==", "!=", ">", "<", ">=", "<="]},
                "value": {"type": "number"}
              }
            }
          },
          "required": ["exit_code"]
        },
        {
          "type": "object",
          "properties": {
            "file_exists": {"type": "string"}
          },
          "required": ["file_exists"]
        },
        {
          "type": "object",
          "properties": {
            "env_set": {"type": "string"}
          },
          "required": ["env_set"]
        },
        {
          "type": "object",
          "properties": {
            "equals": {
              "type": "object",
              "required": ["left", "right"],
              "properties": {
                "left": {"type": "string"},
                "right": {"type": "string"}
              }
            }
          },
          "required": ["equals"]
        },
        {
          "type": "object",
          "properties": {
            "contains": {
              "type": "object",
              "required": ["haystack", "needle"],
              "properties": {
                "haystack": {"type": "string"},
                "needle": {"type": "string"}
              }
            }
          },
          "required": ["contains"]
        },
        {
          "type": "object",
          "properties": {
            "regex": {
              "type": "object",
              "required": ["text", "pattern"],
              "properties": {
                "text": {"type": "string"},
                "pattern": {"type": "string"},
                "flags": {"type": "string"}
              }
            }
          },
          "required": ["regex"]
        },
        {
          "type": "object",
          "properties": {
            "number": {
              "type": "object",
              "required": ["left", "op", "right"],
              "properties": {
                "left": {"type": "string"},
                "op": {"enum": ["==", "!=", ">", "<", ">=", "<="]},
                "right": {"type": "number"}
              }
            }
          },
          "required": ["number"]
        }
      ]
    }
  }
}
```

## State Schema

The state.json file tracks execution progress:

```json
{
  "run_id": "uuid",
  "workflow_name": "demo",
  "status": "running|completed|failed",
  "started_at": "ISO-8601-UTC",
  "current_step": "StepName",
  "context": {},
  "steps": {
    "Process": {
      "status": "completed|failed",
      "exit_code": 0,
      "output": "truncated stdout",
      "duration": 45.2,
      "iterations": [
        {
          "index": 0,
          "item": "file1",
          "status": "completed|failed",
          "exit_code": 0,
          "duration": 1.23,
          "output": "truncated stdout"
        }
      ]
    }
  }
}
```

## Summary of Pivot

| Old "Production" Model | New "Developer" Model |
|------------------------|----------------------|
| Isolated runs in .orchestrator/runs/{run_id} | Workspace-centric operations |
| Read-only workspace/ | Writable workspace/ by default |
| Simple run command, resume for recovery | resume command for failure recovery |
| State file as a run log | State file as a run log |
| Batch-oriented CLI (run, list) | Interactive CLI (watch, run-step, resume) |
| Rigid, controlled execution | Flexible, hackable execution with manual overrides |

## Implementation Guidelines

### Development Phases

**Phase 1: Core Engine (MVP)**
- Basic workflow loading and validation
- Sequential step execution
- Simple variable substitution
- File-based state persistence

**Phase 2: Control Flow**
- Conditional execution (when)
- Branching (on/goto)
- Basic error handling
- Retry logic

**Phase 3: Advanced Features**
- Parallel execution
- For-each and while loops
- Structured conditions
- Provider abstraction

**Phase 4: Production Hardening**
- Schema validation
- Structured logging
- Secret management
- Resource limits
- Idempotency

### Key Design Decisions

1. **File-based over database**: Simplicity and portability
2. **YAML over JSON**: Human readability for workflows
3. **Explicit over implicit**: Clear control flow in V2
4. **Security first**: No shell, no eval, allowlisted environment
5. **Isolation over sharing**: Each run completely isolated

### Testing Strategy

1. **Unit tests**: Each component in isolation
2. **Integration tests**: Complete workflows
3. **Failure tests**: Error conditions and recovery
4. **Performance tests**: Parallel execution and large workflows
5. **Security tests**: Injection attempts and resource exhaustion

### MVP Engine Semantics

**Execution Model**: The MVP engine is strictly single-threaded and sequential. It executes one step at a time in a single process.

**Cancellation**: See Timeout Handling for the only cancellation behavior in MVP.

**Artifact Handling**: Each step writes its artifacts directly to its designated directory (e.g., `workspace/artifacts/{step_name}/`). On retry or resume, any existing artifacts in that directory will be overwritten. The more complex "attempt-based" artifact promotion is deferred.

## MVP Example Workflow

```yaml
version: "1.0"
name: "demo"
strict_flow: true

context:
  project: "my-app"

secrets:
  - CLAUDE_API_KEY

steps:
  - name: Prep
    command: ["python3", "scripts/prep.py", "data.txt"]
    output_file: "prep.txt"
    timeout: 60
    on:
      success:
        goto: Analyze
      failure:
        error: "Prep failed"

  - name: Analyze
    provider: "claude"
    input_file: "prompts/analyze.md"
    output_file: "analysis.txt"
    timeout: 120
    retry:
      attempts: 2
    on:
      success:
        goto: Report
      failure:
        error: "Analysis failed"

  - name: Report
    when:
      step_ok: Analyze
    command: ["python3", "scripts/report.py"]
    input_file: "artifacts/Analyze/analysis.txt"
    output_file: "report.html"
    on:
      success:
        goto: _end
      failure:
        error: "Report generation failed"
```

## MVP Acceptance Tests

1. **Happy path**: Demo workflow creates all artifacts
2. **Timeout**: Step exceeds timeout → killed → exit 124
3. **Retry**: Flaky command succeeds on second attempt
4. **Secrets**: CLAUDE_API_KEY injected, value redacted in logs
5. **Path safety**: Absolute path → error code 3
6. **Resume**: After failure, `orchestrate resume` continues from the last completed step
7. **State tracking**: Logs include clear step start/completion messages
8. **Fresh execution**: Each run executes all steps in order for predictable results
9. **Conditions**: `when.step_ok` gates execution correctly
10. **run-step creates artifacts as expected; prior runs' state.json untouched.**
11. **Substitution in command args**: ${context.x} resolves in command[]; missing var → E_VAR_MISSING.
12. **file_exists base**: Resolves against WORKSPACE; attempts to reference outside BASE → exit 3.

## MVP Implementation Order

1. **Core** (Day 1-2):
   - YAML loader with jsonschema validation
   - State management (read/write state.json)
   - Variable substitution (`${context.*}`, `${steps.*}`)

2. **Execution** (Day 3-4):
   - Sequential step runner
   - Subprocess management with timeout
   - I/O capture and truncation

3. **Conditions** (Day 5):
   - Basic predicates: `step_ok`, `file_exists`, `equals`
   - Boolean combinators: `all`, `any`, `not`

4. **Provider** (Day 6):
   - CLI adapters for claude, openai, gemini
   - Stdin prompt delivery
   - Standardized exit code mapping

5. **CLI** (Day 7):
   - `run` command with context loading
   - Interactive CLI commands (`watch`, `run-step`, `resume`)

6. **Security** (Day 8):
   - Path validation (no absolute, no `..`)
   - Secret injection and redaction
   - Array-only command execution

7. **For-each** (Day 9):
   - Sequential loop execution
   - `_loop_continue` and `_loop_break`

## Success Metrics

**MVP Targets:**
- Run 5-step workflow in < 30 seconds
- Handle 100 sequential steps
- Resume from any failure point
- Zero security violations
- < 500 lines of core code

**Phase 2+ Targets:**
- Execute 10-step workflow in < 1 minute
- Support 100 parallel steps
- Handle 1000-iteration loops
- Recover from transient failures
- < 1000 lines total code

## Final Design Philosophy: The Simple Executor Model

This framework is intentionally designed to provide the simplest, most predictable experience for orchestrating LLM agents.

**What This Model Excels At:**

1. **Zero Magic:** The execution flow is trivial to understand: every step in the workflow is executed in order. There are no complex rules to learn about execution flow.

2. **Perfect for Stochastic Workflows:** This model is built for the reality of LLM agents, where you often want a fresh, non-deterministic result every time you run the workflow.

3. **Effortless Debugging:** When a workflow behaves unexpectedly, the execution flow is straightforward to trace and understand.

4. **Robust and Explicit Recovery:** Time is saved through the deliberate and clear action of using `orchestrate resume`, which provides a reliable way to continue work without ambiguity.

## Phase 3+ Architecture (Future Reference)

### Advanced Engine Semantics

**Scheduler**: The engine will use a work-stealing thread pool with a configurable `max_workers` limit per run. Steps submit tasks to this pool. Child tasks (from `parallel` or `for_each` blocks) inherit a cancellation token from their parent.

**Cancellation**: Cancellation is signaled cooperatively via a dedicated file descriptor (e.g., a readable pipe) passed to each step's subprocess. The subprocess is expected to poll this descriptor. If a step does not terminate within a grace period (e.g., 10s) after signaling, it will be sent `SIGTERM`, followed by `SIGKILL`.

**Artifact Safety**: To prevent corruption during cancellation or retries, each step attempt must write its artifacts to an isolated subdirectory: `workspace/artifacts/{step_name}/{attempt_id}/`. The orchestrator will only promote artifacts from the final, successful attempt's directory to the primary `workspace/artifacts/{step_name}/` location upon successful completion of the step.

**Backpressure**: For steps that stream large amounts of output, the engine will use a 1MB in-memory ring buffer for stdout/stderr. If this buffer overflows, the engine will switch to streaming output directly to a temporary file in the run directory and log that the in-memory output was truncated.
