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
orchestrate run-step <step_name> --workflow workflows/demo.yaml

# Watch for changes and re-run
orchestrate watch workflows/demo.yaml
```

Safety: The `--clean-processed` flag will only operate on the `WORKSPACE/processed/` directory and will refuse to run on any other path. The `--archive-processed` destination path must be outside of the `processed/` directory. If a destination is not provided, the archive defaults to `RUN_ROOT/processed.zip`.

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

### Edge Case Behavior

The following behaviors are **implementation-defined**:

Undefined Variables:
- Implementations may substitute with empty string
- Or leave literal `"${undefined.var}"` in place
- Or raise an error and halt execution

Type Coercion in Conditions:
- String comparison semantics are implementation-defined
- Behavior when comparing JSON numbers to string literals is implementation-defined

Escape Syntax:
- The method to output literal `"${"` in strings is implementation-defined

Recommendations for Portability:

To ensure workflows run consistently across implementations:
- Always initialize variables before use via `context` or prior steps
- Use explicit string values in conditions: `"true"` rather than boolean `true`
- Avoid literal `"${"` in strings
- Document implementation requirements if using edge cases

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

# Command override
command_override: ["claude", "-p", "Custom prompt"]

# Wait for files (blocking primitive for inter-agent communication)
wait_for:
  glob: "inbox/engineer/replies/*.task"  # File pattern to watch
  timeout_sec: 1800                      # Max wait time (default: 300)
  poll_ms: 500                          # Poll interval (default: 500)
  min_count: 1                          # Min files required (default: 1)
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

## Provider Execution Model

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

  - name: CustomProvider
    command_override: ["claude", "-p", "Special prompt", "--model", "claude-opus-4-1-20250805"]
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

## Example: Multi-Agent Inbox Processing

```yaml
name: "multi_agent_feature_dev"
strict_flow: true

providers:
  claude:
    command: ["claude", "-p", "${PROMPT}", "--model", "${model}"]
    defaults:
      model: "claude-sonnet-4-20250514"  # Options: claude-opus-4-1-20250805

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
        - name: ImplementWithClaude
          agent: "engineer"
          provider: "claude"
          provider_params:
            model: "claude-sonnet-4-20250514"
          input_file: "${task_file}"  # task_file is already a full path from find
          output_file: "artifacts/engineer/execution_log_${loop.index}.md"  # Captures STDOUT
          # Note: Claude will create implementation files directly
          
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
          command: ["bash", "-c", "
            echo 'Review impl_${loop.index}.py' > inbox/qa/review_${loop.index}.tmp &&
            mv inbox/qa/review_${loop.index}.tmp inbox/qa/review_${loop.index}.task
          "]

  - name: NoTasks
    command: ["echo", "No pending tasks"]
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

## Acceptance Tests

1. Lines capture: `output_capture: lines` → `steps.X.lines[]` populated
2. JSON capture: `output_capture: json` → `steps.X.json` object available
3. Dynamic for-each: `items_from: "steps.List.lines"` iterates correctly
4. Status schema: Write/read status.json with v1 schema
5. Inbox atomicity: `*.tmp` → `rename()` → visible as `*.task`
6. Processed/failed: Success → `processed/{ts}/`, failure → `failed/{ts}/`
7. No env namespace: `${env.*}` rejected by schema validator
8. Provider templates: Template + defaults + params compose argv correctly
9. Command override: Replaces template entirely
10. Clean processed: `--clean-processed` empties directory
11. Archive processed: `--archive-processed` creates zip on success
12. Pointer Grammar: A workflow with `items_from: "steps.X.json.files"` correctly iterates over the nested `files` array
13. JSON Oversize: A step producing >1 MB of JSON correctly fails with exit code 2
14. JSON Parse Error Flag: The same step from above succeeds if `allow_parse_error: true` is set
15. CLI Safety: `orchestrate run --clean-processed` fails if the processed directory is configured outside WORKSPACE
16. Wait for files: `wait_for` step blocks until matching files appear or timeout
17. Wait timeout: `wait_for` with no matching files exits with code 124 after timeout
18. Wait state tracking: `wait_for` records `files`, `wait_duration`, `poll_count` in state.json

---

## Out of Scope

- Concurrency (sequential only)
- While loops
- Parallel execution blocks
- Complex expression evaluation
- Event-driven triggers

