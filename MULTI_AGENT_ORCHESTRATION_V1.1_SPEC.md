# Workflow programming for LLM agents 

## Executive Summary

This specification defines a workflow orchestration system that executes sequences of commands, including LLM model invocations, in a deterministic order. The system uses YAML to define workflows with branching logic. Filesystem directories serve as task queues for inter-agent communication. Each workflow step can invoke shell commands or language model CLIs (Claude Code or Gemini CLI), capture its output in structured formats (text, lines array, or JSON), and have output files (including those created by agent invocation) registered as read dependencies for subsequent steps. The YAML-based orchestration DSL supports string comparison, conditional branching, and loop constructs.

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

Path Resolution Rule: All user-declared paths remain explicit and resolve against WORKSPACE. No auto-prefixing based on agent.

Path Safety (MVP default):
- Reject absolute paths and any path that contains `..` during validation.
- Follow symlinks, but if the resolved real path escapes WORKSPACE, reject the path.
- Enforce these checks at load time and before filesystem operations.

### Run Identity and State Management

#### Run Identification
- **run_id format**: `YYYYMMDDTHHMMSSZ-<6char>` (timestamp + random suffix)
- **RUN_ROOT**: `.orchestrate/runs/${run_id}` under WORKSPACE
- **State persistence**: All run data stored under RUN_ROOT

#### State File Schema

The state file (`${RUN_ROOT}/state.json`) is the authoritative record of execution:

```json
{
  "schema_version": "1.1.1",
  "run_id": "20250115T143022Z-a3f8c2",
  "workflow_file": "workflows/pipeline.yaml",
  "workflow_checksum": "sha256:abcd1234...",
  "started_at": "2025-01-15T14:30:22Z",
  "updated_at": "2025-01-15T14:35:47Z",
  "status": "running",  // running | completed | failed | suspended
  "context": {
    "key": "value"
  },
  "steps": {
    "StepName": {
      "status": "completed",
      "exit_code": 0,
      "started_at": "2025-01-15T14:30:23Z",
      "completed_at": "2025-01-15T14:30:25Z",
      "duration_ms": 2145,
      "output": "...",
      "truncated": false,
      "debug": {
        "command": ["echo", "hello"],
        "cwd": "/workspace",
        "env_count": 42
      }
    }
  },
  "for_each": {
    "ProcessItems": {
      "items": ["file1.txt", "file2.txt"],
      "completed_indices": [0],
      "current_index": 1
    }
  }
}
```

Step Status Semantics:
- Step `status` values: `pending | running | completed | failed | skipped`.
- Conditional steps with `when` that evaluate false are marked `skipped` and do not execute a process (treat as `exit_code: 0`).

#### State Integrity

**Corruption Detection:**
- State file includes `workflow_checksum` to detect workflow modifications
- Each state update atomically writes to `.state.json.tmp` then renames
- Malformed JSON or schema violations trigger recovery mode

**Recovery Mechanisms:**
```bash
# Resume with state validation
orchestrate resume <run_id>  # Validates and continues

# Force restart ignoring corrupted state
orchestrate resume <run_id> --force-restart  # Creates new state

# Attempt repair of corrupted state
orchestrate resume <run_id> --repair  # Best-effort recovery

# Archive old runs
orchestrate clean --older-than 7d  # Remove old run directories
```

**State Backup (MVP policy):**
- When `--backup-state` is enabled (or in `--debug`), before each step execution copy `state.json` to `state.json.step_${step_name}.bak`.
- Keep last 3 backups per run (rotating).
- On corruption, attempt rollback to last valid backup.

### Task Queue System

Writing Tasks:
1. Create as `*.tmp` file
2. Atomic rename to `*.task`

Processing Results:
- Success: `mv <task> processed/{timestamp}/`
- Failure (non-retryable): `mv <task> failed/{timestamp}/`

File Content: Freeform text; JSON recommended for structured data

Configuration:
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
# Run workflow from beginning
orchestrate run workflows/demo.yaml \
  --context key=value \
  --context-file context.json \
  --clean-processed \           # Empty processed/ before run
  --archive-processed output.zip # Archive processed/ on success

# Resume failed/interrupted run
orchestrate resume <run_id>

# Execute single step
orchestrate run-step <step_name> --workflow workflows/demo.yaml   # Optional (post-MVP)

# Watch for changes and re-run
orchestrate watch workflows/demo.yaml                             # Optional (post-MVP)
```

Safety: The `--clean-processed` flag will only operate on the `WORKSPACE/processed/` directory and will refuse to run on any other path. The `--archive-processed` destination path must be outside of the `processed/` directory. If a destination is not provided, the archive defaults to `RUN_ROOT/processed.zip`.

### Extended CLI Options

#### Debugging and Recovery Flags

```bash
# Debug and observability
--debug                 # Enable debug logging
--progress              # Show real-time progress
--trace                 # Include trace IDs in logs
--dry-run              # Validate without execution

# State management
--force-restart        # Ignore existing state
--repair               # Attempt state recovery
--backup-state         # Force state backup before each step
--state-dir <path>     # Override default .orchestrate/runs

# Error handling
--on-error stop|continue|interactive  # Error behavior (interactive optional/post-MVP)
--max-retries <n>      # Retry failed steps (default: 0)
--retry-delay <ms>     # Delay between retries

# Output control  
--quiet                # Minimal output
--verbose              # Detailed output
--json                 # JSON output for tooling (optional/post-MVP)
--log-level debug|info|warn|error
```

#### Environment Variables

```bash
# Override defaults
ORCHESTRATE_DEBUG=1              # Same as --debug
ORCHESTRATE_STATE_DIR=/tmp/runs # Custom state location
ORCHESTRATE_LOG_LEVEL=debug     # Default log level
ORCHESTRATE_KEEP_RUNS=30         # Days to keep old runs
```

## Variable Model

### Namespaces (precedence order)

1. Run Scope
   - `${run.timestamp_utc}` - The start time of the run, formatted as YYYYMMDDTHHMMSSZ

2. Loop Scope
   - `${item}` - Current iteration value
   - `${loop.index}` - Current iteration (0-based)
   - `${loop.total}` - Total iterations

3. Step Results
   - `${steps.<name>.exit_code}` - Step completion code
   - `${steps.<name>.output}` - Step stdout (text mode)
   - `${steps.<name>.lines}` - Array when `output_capture: lines`
   - `${steps.<name>.json}` - Object when `output_capture: json`
   - `${steps.<name>.duration}` - Execution time

4. Context Variables
   - `${context.<key>}` - Workflow-level variables

### Variable Substitution Scope

Where Variables Are Substituted:
- Command arrays: `["echo", "${context.message}"]`
- File paths: `"artifacts/${loop.index}/result.md"`
- Provider parameters: `model: "${context.model_name}"`
- Conditional values: `left: "${steps.Previous.output}"`
- Dependency paths: `depends_on.required: ["data/${context.dataset}/*.csv"]`

Where Variables Are NOT Substituted:
- File contents: The contents of files referenced by `input_file`, `output_file`, or any other file parameters are passed as-is without variable substitution

Dynamic Content Pattern:

To include dynamic content in files, use a pre-processing step:

```yaml
steps:
  # Step 1: Create dynamic prompt with substituted variables
  - name: PreparePrompt
    command: ["bash", "-c", "echo 'Analyze ${context.project_name}' > temp/prompt.md"]
    
  # Step 2: Use the prepared prompt
  - name: Analyze
    provider: "claude"
    input_file: "temp/prompt.md"  # Contains substituted content
```

Template processing for file contents is not currently supported. Files are passed literally without variable substitution.

### MVP Defaults and Edge Cases

Undefined Variables (MVP default):
- Referencing an undefined variable is an error; the step halts with exit code 2 and records error context. A future flag may allow treating undefined as empty string.

Type Coercion in Conditions (MVP default):
- Conditions compare as strings; JSON numbers and booleans are coerced to strings prior to comparison.

Escape Syntax (MVP default):
- Use `$$` to render a literal `$`. To render the sequence `${`, write `$${`.

Portability Recommendations:
- Initialize variables before use via `context` or prior steps.
- Prefer explicit strings in conditions: `"true"` rather than boolean `true`.
- Avoid literal `${` in strings unless escaped as `$${`.

### Environment & Secrets

Per-step environment injection (not substitution):
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

## Step Schema

### Fields

```yaml
# Agent label (optional)
agent: "engineer"  # Informational label only; doesn't affect path resolution

# Output capture mode
output_capture: "text"  # Default: text | lines | json

# Dynamic for-each from prior step
for_each:
  items_from: "steps.CheckInbox.lines"  # Reference array from prior step
  # OR literal array:
  items: ["a", "b", "c"]
  as: item
  steps: [...]

# Optional, only applies when output_capture: json
allow_parse_error: false

# Provider specification
provider: "claude"
provider_params:
  model: "claude-sonnet-4-20250514"  # Options: claude-opus-4-1-20250805

# Raw command (mutually exclusive with provider)
command: ["claude", "-p", "Custom prompt"]

# Wait for files (blocking primitive for inter-agent communication)
wait_for:
  glob: "inbox/engineer/replies/*.task"  # File pattern to watch
  timeout_sec: 1800                      # Max wait time (default: 300)
  poll_ms: 500                          # Poll interval (default: 500)
  min_count: 1                          # Min files required (default: 1)
  # Note: wait_for is mutually exclusive with command/provider/for_each in the same step.

# Dependency validation and injection
depends_on:
  required:   # Files that must exist
    - "config/*.yaml"
    - "data/${context.dataset}/*.csv"
  optional:   # Files that may exist
    - "cache/previous.json"
  inject: true  # Auto-inject file list
  # OR advanced form:
  inject:
    mode: "list"  # How to inject: list | content | none
    instruction: "Custom instruction for these files:"
    position: "prepend"  # Where to inject: prepend | append
```

Pointer Syntax: The value must be a string in the format `steps.<StepName>.lines` or `steps.<StepName>.json[.<dot.path>]`. The referenced value must resolve to an array. Dot-paths do not support wildcards or advanced expressions.

### Output Capture Modes

`text` (default): Traditional string capture
```json
{
  "output": "First line\nSecond line\n",
  "truncated": false
}
```

`lines`: Split into array of lines
```json
{
  "output_capture": "lines",
  "lines": ["inbox/engineer/task1.task", "inbox/engineer/task2.task"],
  "truncated": false
}
```

`json`: Parse as JSON object
```json
{
  "output_capture": "json",
  "json": {"success": true, "files": ["a.py", "b.py"]},
  "truncated": false
}
```

Parse failure → exit code 2 unless `allow_parse_error: true`

Limits:
- text: The first 8 KB of stdout is stored in the state file. If the stream exceeds 1 MB, it is spilled to a log file and the `output` field is marked as truncated.
- lines: A maximum of 10,000 lines are stored. If exceeded, the `lines` array will contain the first 10,000 entries and a `truncated: true` flag will be set.
- json: The orchestrator will buffer up to 1 MB of stdout for parsing. If stdout exceeds this limit, parsing fails with exit code 2 (unless `allow_parse_error: true`). Invalid JSON also results in exit code 2.

State Fields: When `output_capture` is set to `lines` or `json`, the raw `output` field is omitted from the step's result in `state.json` to avoid data duplication.

### Control Flow Defaults (MVP)
- `strict_flow: true` means any non-zero exit halts the run unless an `on.failure.goto` is defined for that step.
- `_end` is a reserved `goto` target that terminates the run successfully.

## Provider Execution Model

### Command Construction and ${PROMPT}

Mutual exclusivity (MVP rule):
- A step must specify either `provider` (with optional `provider_params`) or a raw `command`, but not both. Using both is a validation error.
- The deprecated `command_override` field is not supported; use a raw `command` step instead for fully manual invocations.

Execution:
- If `provider` is set, use the provider template with merged parameters (`defaults` overridden by `provider_params`).
- If `command` is set (and no `provider`), execute the raw command array as-is.

Reserved placeholder `${PROMPT}`:
- Provider templates may include `${PROMPT}` to receive the composed input text as a single argv token.
- `${PROMPT}` is built from `input_file` contents after optional dependency injection. Injection is performed in-memory; input files are never modified on disk.

### Input/Output Contract

When a step specifies both `provider` and `input_file`:

1. Input Handling: The orchestrator reads `input_file` contents and passes them as a command-line argument to the provider
2. Output Handling: If `output_file` is specified, STDOUT is redirected to that file
3. File contents passed literally: The prompt text is passed as a CLI argument, not piped via STDIN

Example Execution:
```yaml
steps:
  - name: Analyze
    provider: "claude"
    input_file: "prompts/analyze.md"
    output_file: "artifacts/analysis.md"

# Orchestrator reads the file:
PROMPT_CONTENT=$(cat prompts/analyze.md)

# Then executes:
claude -p "$PROMPT_CONTENT" --model claude-sonnet-4-20250514 > artifacts/analysis.md
#      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#      Prompt text as CLI arg + model selection
```

Prompt Contents:
```markdown
# prompts/analyze.md
Analyze the system requirements and create a detailed architecture.
Consider scalability, security, and maintainability.
Read any files in artifacts/requirements/ for context.
```

Key Benefits:
- No duplication: Prompt doesn't need to reference itself
- Natural prompts: Write prompts as you normally would
- Provider flexibility: Provider can still read other files mentioned IN the prompt
- Clean separation: `input_file` = the prompt, other files = data the prompt references

Output capture with `output_file`:
- When `output_file` is set, STDOUT is redirected to that file. The orchestrator still captures output per `output_capture` rules into state (subject to truncation/spill rules) for observability.

### Provider File Operations

### Concurrent File and Stream Output

Providers can read and write files directly from/to the filesystem while also outputting to STDOUT. These capabilities coexist:

1. Direct File Operations: Providers may create, read, or modify files anywhere in the workspace based on prompt instructions
2. STDOUT Capture: The `output_file` parameter captures STDOUT (typically logs, status messages, or reasoning process)
3. Simultaneous Operation: A provider invocation may write multiple files AND produce STDOUT output

Example:
```yaml
steps:
  - name: GenerateSystem
    agent: "architect"
    provider: "claude"
    input_file: "prompts/design.md"
    output_file: "artifacts/architect/execution_log.md"  # Captures STDOUT
    # Provider may also create files directly:
    # - artifacts/architect/system_design.md
    # - artifacts/architect/api_spec.md
    # - artifacts/architect/data_model.md
```

### Best Practices

- Use `output_file` to capture execution logs and agent reasoning for debugging
- Design prompts to write primary outputs as files to appropriate directories
- Use subsequent steps to discover and validate created files
- Document expected file outputs in step comments for clarity

### File Dependencies

#### Optional Dependency Declaration

Steps may declare file dependencies for pre-execution validation and documentation:

**Syntax:**
```yaml
steps:
  - name: ProcessData
    provider: "claude"
    input_file: "prompts/process.md"
    depends_on:
      required:  # Must exist or step fails
        - "config.json"
        - "data/*.csv"
        - "artifacts/architect/*.md"
      optional:  # Warning if missing, continues execution
        - "cache/previous.json"
        - "docs/reference.md"
```

#### Behavior Rules

1. **Validation Time**: Immediately before step execution (after previous steps complete)
2. **Variable Substitution**: `${...}` variables ARE substituted using current context
3. **Pattern Syntax**: POSIX glob patterns (`*` and `?` wildcards); globstar `**` is not supported in v1.1
4. **Pattern Matching**: 
   - `required` + 0 matches = exit code 2 (non-retryable failure)
   - `optional` + 0 matches = no warning (intentional)
5. **Existence Check**: File OR directory present = exists
6. **Path Resolution**: Relative to WORKSPACE
7. **Symlinks**: Followed
8. **Loop Context**: Re-evaluated each iteration with current loop variables

#### Error Handling

- Missing required dependency produces exit code 2 (non-retryable)
- Standard `on.failure` handlers apply
- Error message includes the missing path/pattern

#### Benefits

- **Fail Fast**: Detect missing files before expensive provider calls
- **Documentation**: Explicit declaration of file relationships  
- **Better Errors**: Clear messages about what's missing
- **Future-Ready**: Enables caching and change detection in v2

#### Dependency Injection

The orchestrator can automatically inject dependency information into the provider's input, eliminating the need to manually maintain file lists in prompts.

##### Basic Injection

```yaml
depends_on:
  required:
    - "artifacts/architect/*.md"
  inject: true  # Enable auto-injection with defaults
```

When `inject: true`, the orchestrator prepends:
```
The following files are required inputs for this task:
- artifacts/architect/system_design.md
- artifacts/architect/api_spec.md

[original prompt content]
```

##### Advanced Injection

```yaml
depends_on:
  required:
    - "artifacts/architect/*.md"
  optional:
    - "docs/standards.md"
  inject:
    mode: "list"  # list | content | none (default: none)
    instruction: "Review these architecture files:"  # Custom instruction
    position: "prepend"  # prepend | append (default: prepend)
```

##### Injection Modes

**`list` mode**: Injects file paths with instruction
```
Review these architecture files:
Required:
- artifacts/architect/system_design.md
- artifacts/architect/api_spec.md
Optional (if available):
- docs/standards.md

[original prompt content]
```

**`content` mode**: Injects full file contents
```
Review these architecture files:

=== File: artifacts/architect/system_design.md ===
[full content of file]

=== File: artifacts/architect/api_spec.md ===
[full content of file]

[original prompt content]
```

**`none` mode** (default): No injection, manual coordination required

##### Default Instructions

If `instruction` is not specified, the orchestrator uses context-appropriate defaults:

- **list mode**: "The following files are required inputs for this task:"
- **content mode**: "The following file contents are provided for context:"

##### Injection Rules

1. **Pattern Resolution**: Glob patterns are resolved to actual file paths before injection
2. **Variable Substitution**: Variables in paths are substituted before injection
3. **Missing Optional Files**: Omitted from injection without error
4. **Empty Required Patterns**: If a required pattern matches no files, dependency validation fails (no injection occurs)
5. **Position**: 
   - `prepend`: Injection appears before original prompt content
   - `append`: Injection appears after original prompt content
6. **Injection Target**: Injection modifies the in-memory prompt passed to the provider; the input_file on disk is not changed

##### Size Limits and Truncation

**Rationale:** The 256 KiB cap prevents memory exhaustion while accommodating typical use cases (100+ file paths or several configuration files).

**Truncation Behavior:**

In **list mode:**
```
Review these files:
- file1.md
- file2.md
...
- file99.md
[... 47 more files truncated (312 KiB total)]
```

In **content mode:**
```
=== File: config.yaml (45 KiB) ===
[content]

=== File: data.json (180 KiB) ===
[partial content]
[... truncated: 125 KiB of 180 KiB shown]

=== Files not shown (5 files, 892 KiB) ===
- archived/old1.json (201 KiB)
- archived/old2.json (189 KiB)
[... 3 more files]
```

**Truncation Record:**
When truncation occurs, state.json records:
```json
{
  "injection_truncated": true,
  "truncation_details": {
    "total_size": 524288,
    "shown_size": 262144,
    "files_shown": 2,
    "files_truncated": 1,
    "files_omitted": 5
  }
}
```

#### Migration Notes

##### From v1.1 to v1.1.1
Existing workflows with `depends_on` continue to work unchanged. To adopt injection:

**Before (v1.1):**
```yaml
# Had to maintain file list in both places
depends_on:
  required:
    - "artifacts/architect/*.md"
input_file: "prompts/implement.md"  # Must list files manually
```

**After (v1.1.1):**
```yaml
# Single source of truth
depends_on:
  required:
    - "artifacts/architect/*.md"
  inject: true  # Automatically informs provider
input_file: "prompts/generic_implement.md"  # Can be generic
```

##### Benefits of Injection
- **DRY Principle**: Declare files once in YAML
- **Pattern Support**: `*.md` expands automatically
- **Maintainability**: Change file lists in one place
- **Flexibility**: Generic prompts work across projects

---

## Provider Configuration

### Direct CLI Integration

The providers are Claude Code (`claude`), Gemini CLI (`gemini`), and similar tools - not raw API calls.

Workflow-level templates:
```yaml
providers:
  claude:
    command: ["claude", "-p", "${PROMPT}", "--model", "${model}"]
    defaults:
      model: "claude-sonnet-4-20250514"  # Options: claude-opus-4-1-20250805
  
  gemini:
    command: ["gemini", "-p", "${PROMPT}"]
    # Gemini CLI doesn't support model selection via CLI
```

Step-level usage:
```yaml
steps:
  - name: Analyze
    provider: "claude"
    provider_params:
      model: "claude-3-5-sonnet"  # or claude-3-5-haiku for faster/cheaper
    input_file: "prompts/analyze.md"
    output_file: "artifacts/architect/analysis.md"

  - name: ManualCommand
    command: ["claude", "-p", "Special prompt", "--model", "claude-opus-4-1-20250805"]
```

Claude Code is invoked with `claude -p "prompt" --model <model>`. Available models:
- `claude-sonnet-4-20250514` - balanced performance
- `claude-opus-4-1-20250805` - most capable

Model can also be set via `ANTHROPIC_MODEL` environment variable or `claude config set model`.

Exit code mapping:
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
      "file": "inbox/qa/review_implementation.task"
    }
  ],
  "message": "Feature implemented successfully"
}
```

Default path: `artifacts/<agent>/status.json` or `status_<step>.json`

Path Convention: All file paths included within a status JSON file (e.g., in the `outputs` array) must be relative to the WORKSPACE directory.

## Debugging and Observability

### Debug Mode

Enable comprehensive logging with `--debug` flag:

```bash
orchestrate run workflow.yaml --debug
```

Debug mode captures:
- Variable substitution traces
- Dependency resolution details  
- Command construction logs
- Environment variable snapshot
- File operation details

### Execution Logs

Each run creates structured logs:

```
${RUN_ROOT}/
├── state.json           # Authoritative state
├── logs/
│   ├── orchestrator.log # Main execution log
│   ├── StepName.stdout  # Step stdout (if >8KB or JSON parse error)
│   ├── StepName.stderr  # Step stderr (always if non-empty)
│   └── StepName.debug   # Debug info (if --debug)
└── artifacts/          # Step-created files
```

### Error Context

On step failure, the orchestrator records:

```json
{
  "error": {
    "message": "Command failed with exit code 1",
    "exit_code": 1,
    "stdout_tail": ["last", "10", "lines"],
    "stderr_tail": ["error", "messages"],
    "context": {
      "undefined_vars": ["${context.missing}"],
      "failed_deps": ["data/required.csv"],
      "substituted_command": ["cat", "data/file_20250115.csv"]
    }
  }
}
```

### Observability Features

**Progress Reporting:**
```bash
# Real-time progress
orchestrate run workflow.yaml --progress

# Output:
[2/5] ProcessData: Running (15s)...
  └─ for_each: [3/10] Processing item3.csv
```

**Metrics Collection:**
State file includes timing for performance analysis:
- Step duration
- Queue wait time (for wait_for steps)
- Provider execution time
- File I/O operations

**Trace Context:**
Each step can emit trace IDs for correlation with external systems:
```yaml
steps:
  - name: CallAPI
    env:
      TRACE_ID: "${run.id}-${steps.index}"
```

## Example: Multi-Agent Inbox Processing

```yaml
version: "1.1"
name: "multi_agent_feature_dev"
strict_flow: true

providers:
  claude:
    command: ["claude", "-p", "${PROMPT}", "--model", "${model}"]
    defaults:
      model: "claude-sonnet-4-20250514"  # Options: claude-opus-4-1-20250805

steps:
  # Architect creates design documents
  - name: ArchitectDesign
    agent: "architect"
    provider: "claude"
    input_file: "prompts/architect/design_system.md"
    output_file: "artifacts/architect/design_log.md"  # Captures thinking
    # Claude creates: artifacts/architect/system_design.md, api_spec.md
    
  # Check what architect created
  - name: ValidateArchitectOutput
    command: ["test", "-f", "artifacts/architect/system_design.md"]
    on:
      failure:
        goto: ArchitectFailed
        
  # Check for engineer tasks in inbox
  - name: CheckEngineerInbox
    command: ["find", "inbox/engineer", "-name", "*.task", "-type", "f"]
    output_capture: "lines"
    on:
      success:
        goto: ProcessEngineerTasks
      failure:
        goto: CreateEngineerTasks
        
  # Create tasks from architect output
  - name: CreateEngineerTasks
    command: ["bash", "-c", "
      echo 'Implement the system described in:' > inbox/engineer/implement.tmp &&
      ls artifacts/architect/*.md >> inbox/engineer/implement.tmp &&
      mv inbox/engineer/implement.tmp inbox/engineer/implement.task
    "]
    on:
      success:
        goto: CheckEngineerInbox
        
  # Process each engineer task
  - name: ProcessEngineerTasks
    for_each:
      items_from: "steps.CheckEngineerInbox.lines"
      as: task_file
      steps:
        - name: ImplementWithClaude
          agent: "engineer"
          provider: "claude"
          input_file: "prompts/engineer/generic_implement.md"  # Now can be generic!
          output_file: "artifacts/engineer/impl_log_${loop.index}.md"
          depends_on:
            required:  # Must have architect's designs
              - "artifacts/architect/system_design.md"
              - "artifacts/architect/api_spec.md"
            optional:  # Nice to have if available
              - "docs/coding_standards.md"
              - "artifacts/architect/examples.md"
            inject:
              mode: "list"
              instruction: "Implement the system based on these architecture documents:"
          on:
            failure:
              goto: HandleMissingDependencies
          # Claude writes: src/impl_${loop.index}.py
          
        - name: WriteStatus
          command: ["echo", '{"success": true, "task": "${task_file}", "impl": "src/impl_${loop.index}.py"}']
          output_file: "artifacts/engineer/status_${loop.index}.json"
          output_capture: "json"
          
        - name: MoveToProcessed
          command: ["bash", "-c", "mkdir -p processed/${run.timestamp_utc}_${loop.index} && mv ${task_file} processed/${run.timestamp_utc}_${loop.index}/"]
          
        - name: CreateQATask
          when:
            equals:
              left: "${steps.WriteStatus.json.success}"
              right: "true"
          command: ["echo", "Review src/impl_${loop.index}.py from ${task_file}"]
          output_file: "inbox/qa/review_${loop.index}.task"
          depends_on:
            required:
              - "src/impl_${loop.index}.py"  # Verify engineer created it
              
  # Error handlers
  - name: ArchitectFailed
    command: ["echo", "ERROR: Architect did not create required design files"]
    on:
      success:
        goto: _end
        
  - name: HandleMissingDependencies  
    command: ["echo", "ERROR: Required architect artifacts missing for engineer"]
    on:
      success:
        goto: _end
```

---

## Prompt Management Patterns

### Directory Purpose Clarification

- **`prompts/`**: Static, reusable prompt templates created by workflow authors before execution
- **`inbox/`**: Dynamic task files for agent coordination, created during workflow execution
- **`temp/`**: Temporary files for dynamic prompt composition and intermediate processing

### Multi-Agent Coordination Pattern

When agent B needs to process outputs from agent A:

```yaml
steps:
  # Step 1: Agent A creates artifacts
  - name: ArchitectDesign
    agent: "architect"
    provider: "claude"
    input_file: "prompts/architect/design.md"  # Contains prompt text
    output_file: "artifacts/architect/log.md"   # Captures STDOUT
    
    # prompts/architect/design.md contains:
    # "Create a system architecture. Read requirements from artifacts/requirements/*.md
    #  Write your design to artifacts/architect/system_design.md and api_spec.md"
    
    # Executes as:
    # claude -p "Create a system architecture..." --model claude-sonnet-4-20250514 > artifacts/architect/log.md
    
  # Step 2: Compose dynamic task for Agent B (atomic write)
  - name: PrepareEngineerTask
    command: ["bash", "-c", "
      echo 'Implement the following designs:' > inbox/engineer/task_${run.timestamp_utc}.tmp &&
      echo '' >> inbox/engineer/task_${run.timestamp_utc}.tmp &&
      cat artifacts/architect/*.md >> inbox/engineer/task_${run.timestamp_utc}.tmp &&
      mv inbox/engineer/task_${run.timestamp_utc}.tmp inbox/engineer/task_${run.timestamp_utc}.task
    "]
    
  # Step 3: Agent B processes dynamic task
  - name: EngineerImplement
    agent: "engineer"
    provider: "claude"
    input_file: "inbox/engineer/task_${run.timestamp_utc}.task"  # Dynamically composed
```

### Best Practices

- Keep static templates generic (reference concepts, not specific files)
- Build dynamic prompts at runtime to reference actual artifacts
- Use `inbox/` for agent work queues to maintain clear task boundaries
- Document the expected flow between agents in workflow comments

---

## Example: File Dependencies in Complex Workflows

This example demonstrates all dependency features including patterns, variables, and loops:

```yaml
version: "1.1"
name: "data_pipeline_with_dependencies"

context:
  dataset: "customer_2024"
  model_version: "v3"

steps:
  # Validate all input data exists
  - name: ValidateInputs
    command: ["echo", "Checking inputs..."]
    depends_on:
      required:
        - "config/pipeline.yaml"
        - "data/${context.dataset}/*.csv"  # Variable substitution
        - "models/${context.model_version}/weights.pkl"
      optional:
        - "cache/${context.dataset}/*.parquet"
    
  # Process each CSV file
  - name: ProcessDataFiles
    command: ["find", "data/${context.dataset}", "-name", "*.csv"]
    output_capture: "lines"
    
  - name: TransformFiles
    for_each:
      items_from: "steps.ProcessDataFiles.lines"
      as: csv_file
      steps:
        - name: ValidateAndTransform
          provider: "data_processor"
          input_file: "${csv_file}"
          output_file: "processed/${loop.index}.parquet"
          depends_on:
            required:
              - "${csv_file}"  # Current iteration file
              - "config/transformations.yaml"
            optional:
              - "processed/${loop.index}.cache"  # Previous run cache
              
        - name: GenerateReport
          provider: "claude"
          input_file: "prompts/analyze_data.md"
          output_file: "reports/analysis_${loop.index}.md"
          depends_on:
            required:
              - "processed/${loop.index}.parquet"  # From previous step
              
  # Final aggregation needs all processed files
  - name: AggregateResults
    provider: "aggregator"
    input_file: "prompts/aggregate.md"
    output_file: "reports/final_report.md"
    depends_on:
      required:
        - "processed/*.parquet"  # Glob pattern for all outputs
      optional:
        - "reports/analysis_*.md"  # All analysis reports
```

---

## Example: Dependency Injection Modes

```yaml
version: "1.1"
name: "injection_modes_demo"

steps:
  # Simple injection with defaults
  - name: SimpleReview
    provider: "claude"
    input_file: "prompts/generic_review.md"
    depends_on:
      required:
        - "src/*.py"
      inject: true  # Uses default instruction
    
  # List mode with custom instruction
  - name: ImplementFromDesign
    provider: "claude"
    input_file: "prompts/implement.md"
    depends_on:
      required:
        - "artifacts/architect/*.md"
      inject:
        mode: "list"
        instruction: "Your implementation must follow these design documents:"
        position: "prepend"
    
  # Content mode for data processing
  - name: ProcessJSON
    provider: "data_processor"
    input_file: "prompts/transform.md"
    depends_on:
      required:
        - "data/input.json"
      inject:
        mode: "content"
        instruction: "Transform this JSON data according to the rules below:"
        position: "prepend"
    # The actual JSON content is injected before the transformation rules
    
  # Append mode for context
  - name: GenerateReport
    provider: "claude"
    input_file: "prompts/report_template.md"
    depends_on:
      optional:
        - "data/statistics/*.csv"
      inject:
        mode: "content"
        instruction: "Reference data for your report:"
        position: "append"
    # Template comes first, data is appended as reference
    
  # Pattern expansion with injection (non-recursive)
  - name: ReviewAllCode
    provider: "claude"
    input_file: "prompts/code_review.md"
    depends_on:
      required:
        - "src/*.py"      # POSIX glob; non-recursive
        - "tests/*.py"
      inject:
        mode: "list"
        instruction: "Review all these Python files for quality and consistency:"
    # All matched files are listed with clear instruction
    # Note: For recursive discovery, first generate a file list via a step like
    #   command: ["bash", "-c", "find src tests -name '*.py' -type f"] with output_capture: lines,
    # then pass that list through a for_each loop or render into a prompt file.
    
  # No injection (classic mode)
  - name: ManualCoordination
    provider: "claude"
    input_file: "prompts/specific_files_mentioned.md"
    depends_on:
      required:
        - "data/important.csv"
      inject: false  # Or omit inject entirely
    # Prompt must manually reference the files
```

---

## Example: Debugging Failed Runs

```yaml
version: "1.1"
name: "debugging_example"

steps:
  - name: ProcessWithDebug
    command: ["python", "process.py"]
    env:
      DEBUG: "${ORCHESTRATE_DEBUG:-0}"  # Inherit debug flag
    on:
      failure:
        goto: DiagnoseFailure
        
  - name: DiagnoseFailure
    command: ["bash", "-c", "
      echo 'Checking failure context...' &&
      cat ${RUN_ROOT}/logs/ProcessWithDebug.stderr &&
      jq '.steps.ProcessWithDebug.error' ${RUN_ROOT}/state.json
    "]
```

### Investigating Failures

```bash
# Run with debugging
orchestrate run workflow.yaml --debug

# On failure, check logs
cat .orchestrate/runs/latest/logs/orchestrator.log

# Examine state
jq '.steps | map_values({status, exit_code, error})' \
  .orchestrate/runs/latest/state.json

# Resume after fixing issue
orchestrate resume 20250115T143022Z-a3f8c2 --debug
```

---

## Acceptance Tests

1. Lines capture: `output_capture: lines` → `steps.X.lines[]` populated
2. JSON capture: `output_capture: json` → `steps.X.json` object available
3. Dynamic for-each: `items_from: "steps.List.lines"` iterates correctly
4. Status schema: Write/read status.json with v1 schema
5. Inbox atomicity: `*.tmp` → `rename()` → visible as `*.task`
6. Processed/failed: Success → `processed/{ts}/`, failure → `failed/{ts}/`
7. No env namespace: `${env.*}` rejected by schema validator
8. Provider templates: Template + defaults + params compose argv correctly
9. Provider/Command exclusivity: Validation error when a step includes both `provider` and `command`
10. Clean processed: `--clean-processed` empties directory
11. Archive processed: `--archive-processed` creates zip on success
12. Pointer Grammar: A workflow with `items_from: "steps.X.json.files"` correctly iterates over the nested `files` array
13. JSON Oversize: A step producing >1 MB of JSON correctly fails with exit code 2
14. JSON Parse Error Flag: The same step from above succeeds if `allow_parse_error: true` is set
15. CLI Safety: `orchestrate run --clean-processed` fails if the processed directory is configured outside WORKSPACE
16. Wait for files: `wait_for` step blocks until matching files appear or timeout
17. Wait timeout: `wait_for` with no matching files exits with code 124 after timeout
18. Wait state tracking: `wait_for` records `files`, `wait_duration`, `poll_count` in state.json
19. **Dependency Validation:** Step with `depends_on.required: ["missing.txt"]` fails with exit code 2
20. **Dependency Patterns:** `depends_on.required: ["*.csv"]` correctly matches files using POSIX glob
21. **Variable in Dependencies:** `depends_on.required: ["${context.file}"]` substitutes variable before validation
22. **Loop Dependencies:** Dependencies re-evaluated each iteration with current loop context
23. **Optional Dependencies:** Missing optional dependencies log warning but continue execution
24. **Dependency Error Handler:** `on.failure` catches dependency validation failures (exit code 2)
25. **Basic Injection:** Step with `inject: true` prepends default instruction and file list
26. **List Mode Injection:** `inject.mode: "list"` correctly lists all resolved file paths
27. **Content Mode Injection:** `inject.mode: "content"` includes full file contents
28. **Custom Instruction:** `inject.instruction` replaces default instruction text
29. **Append Position:** `inject.position: "append"` places injection after prompt content
30. **Pattern Injection:** Glob patterns resolve to full file list before injection
31. **Optional File Injection:** Missing optional files omitted from injection without error
32. **No Injection Default:** Without `inject` field, no modification to prompt occurs
33. **Wait-For Exclusivity:** A step that combines `wait_for` with `command`/`provider`/`for_each` is rejected by validation
34. **Conditional Skip:** When `when.equals` evaluates false, step `status` is `skipped` with `exit_code: 0`
35. **Path Safety (Absolute):** Absolute paths are rejected at validation time
36. **Path Safety (Parent Escape):** Paths containing `..` or symlinks resolving outside WORKSPACE are rejected
37. **Deprecated override:** Using `command_override` is rejected by the schema/validator; authors must use `command` for manual invocations

---

## Out of Scope

- Concurrency (sequential only)
- While loops
- Parallel execution blocks
- Complex expression evaluation
- Event-driven triggers
