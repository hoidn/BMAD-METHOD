# Multi-Agent Orchestrator v1.1 — Implementation Architecture

Document version: 1.0
Target spec: MULTI_AGENT_ORCHESTRATION_V1.1_SPEC.md
Implementation target: Node.js (>=20), plain JS with JSDoc types or TypeScript equivalent

## 1) Goals & Non‑Goals

- Goals
  - Execute YAML-defined workflows deterministically and sequentially.
  - Support branching (`on.success`/`on.failure`) and simple conditionals (`when.equals`).
  - Support for-each loops driven by previous step outputs (`items_from` pointers) or literal arrays.
  - Integrate directly with provider CLIs (e.g., Claude Code) via argv composition.
  - File-based inbox/processed/failed queues for inter-agent communication.
  - Persist authoritative state under `.orchestrate/runs/<run_id>/state.json` with atomic updates and backups.
  - Robust debug/observability, safe flags (`--clean-processed`, `--archive-processed`).

- Non-goals (per spec “Out of Scope”)
  - Concurrency/parallel blocks, while-loops, complex expressions.
  - Event-driven triggers beyond simple polling (`wait_for`).


## 2) Workspace & Runtime Layout

- Workspace (all paths relative to WORKSPACE)
  - `src/` user code
  - `prompts/` reusable templates
  - `artifacts/{architect,engineer,qa}/` agent outputs
  - `inbox/{architect,engineer,qa}/` file queues (`*.task`)
  - `processed/{ts}/` completed tasks
  - `failed/{ts}/` failed tasks

- Run state directory
  - `RUN_ROOT = .orchestrate/runs/<run_id>`
  - `state.json` (authoritative)
  - `logs/` (orchestrator.log, StepName.{stdout,stderr,debug})
  - `artifacts/` (optional mirror of step spills if needed)

- Path resolution rule: Use literal paths provided in workflow; resolve against WORKSPACE; no agent-based auto-prefixing.

## 2a) Architectural Decisions (ADRs)

- ADR-01 Filesystem as Source of Truth and Path Safety
  - All state, artifacts, and inter-agent communication live in the filesystem under WORKSPACE.
  - Reject absolute paths and any path containing `..` during validation.
  - Follow symlinks, but if the resolved real path would escape WORKSPACE, reject the path.
  - Enforce these rules at load/validation time and before any filesystem operation.

- ADR-02 Declarative YAML, Sequential Execution
  - YAML defines the workflow; the engine executes one step at a time, deterministically.

- ADR-03 Provider as Managed Black Box (with assumed side effects)
  - Explicit contract: orchestrator constructs argv, passes prompt as a single CLI argument, injects env/secrets, captures stdout/stderr/exit code.
  - Implicit contract: providers may read/write workspace files as side effects; subsequent steps and `depends_on` validate these effects.

- ADR-04 Authoritative JSON Run State
  - `state.json` is authoritative; update after every step attempt to support resumability and auditability.


## 3) High-level System Architecture

- CLI Layer (`orchestrate`)
  - Commands: `run`, `resume`, `run-step`, `watch`, `clean` (archive/remove old runs)
  - Flags: debug, progress, trace, dry-run, force-restart, repair, backup-state, state-dir, on-error, max-retries, retry-delay, quiet/verbose/json/log-level, clean-processed, archive-processed

- Orchestration Engine
  - Loads & validates workflow YAML; computes checksum.
  - Executes steps sequentially with branching and simple conditions.
  - Manages variable substitution, loop scope, pointer resolution, and step outputs.

- State Manager
  - Run ID creation, RUN_ROOT setup, atomic state writes (tmp + rename), rotating backups, corruption detection, repair/rollback, checksum enforcement.

- Step Executor
  - Builds argv from provider templates or `command_override`.
  - Injects env + secrets (masked in logs), sets cwd, timeouts, retries.
  - Captures stdout/stderr with modes: text, lines, json; handles truncation/spillover.

- Provider Registry
  - Named providers with command array templates and defaults.
  - Simple substitutions for `PROMPT` (input file content, post-injection) and `provider_params`.

- Dependency Resolver & Injector
  - Validates `depends_on.required|optional` globs with variable substitution.
  - On request, injects dependency info (list or content) into prompt content at prepend/append.

- Wait/Poll Subsystem
  - Implements `wait_for` file polling; records wait metrics.

- Queue Manager
  - Atomic task writes (tmp → rename to `*.task`), moves to processed/failed.
  - Safety rails for `--clean-processed` and `--archive-processed`.

- Observability
  - Structured logs, progress UI, step debug artifacts, JSON output mode.


## 4) Module Structure (suggested)

```
src/
  cli/
    index.ts                 # commander wiring; routes to handlers
    commands/
      run.ts
      resume.ts
      runStep.ts
      watch.ts
      clean.ts
  config/
    types.ts                 # CLI options, global config
    defaults.ts
  workflow/
    loader.ts                # YAML load + schema validation + checksum
    types.ts                 # Workflow, Step, Provider, Condition, DependsOn
    pointers.ts              # steps.X.lines / steps.X.json.path resolution
    substitution.ts          # variable interpolation (context/run/loop/steps)
  state/
    runState.ts              # in-memory model + read/write/backup/repair
    persistence.ts           # atomic write helpers, checksum
  exec/
    runner.ts                # low-level process spawn, env/secrets, timeouts
    stepExecutor.ts          # command construction, uses runner + outputCapture
    outputCapture.ts         # text/lines/json parse, truncation, spill
    retry.ts                 # retry policy helpers
  providers/
    registry.ts              # register/get provider templates
    types.ts                 # ProviderSpec, ProviderParams
  deps/
    resolver.ts              # glob resolve, required/optional, errors
    injector.ts              # list/content injection composition
  fsq/
    queue.ts                 # inbox read/write, processed/failed moves
    wait.ts                  # wait_for polling with timeout
  logging/
    logger.ts                # levels, masking, trace IDs, progress
  observe/
    status.ts                # status JSON writer/validator
  watch/
    watcher.ts               # rerun on file changes (if implemented)
  utils/
    fs.ts, glob.ts, time.ts, json.ts, zip.ts, mask.ts
```


## 5) Data Models (TypeScript-style)

### 5.1 Workflow spec

```ts
type OutputCapture = 'text' | 'lines' | 'json';

interface ProviderParams { [k: string]: string; }

interface ProviderTemplate {
  name: string;                 // e.g., 'claude'
  command: string[];            // argv template e.g., ["claude","-p","${PROMPT}","--model","${model}"]
  defaults?: ProviderParams;    // e.g., { model: 'claude-sonnet-4-20250514' }
}

interface DependsOnConfigBasic {
  required?: string[];          // globs (after var-substitution)
  optional?: string[];
  inject?: boolean | DependsOnInjection;
}

interface DependsOnInjection {
  mode?: 'list' | 'content' | 'none';   // default: none
  instruction?: string;                 // default text per spec
  position?: 'prepend' | 'append';      // default: prepend
}

interface WaitForConfig {
  glob: string;
  timeout_sec?: number;   // default 300
  poll_ms?: number;       // default 500
  min_count?: number;     // default 1
}

interface ForEachBlock {
  items_from?: string;    // pointer e.g., 'steps.Check.lines' or 'steps.X.json.arr'
  items?: string[];       // literal
  as?: string;            // variable name (default: 'item')
  steps: Step[];          // nested steps
}

interface ConditionEquals {
  equals: { left: string; right: string };
}

interface StepBase {
  name: string;
  agent?: string;                         // label only
  when?: ConditionEquals;                 // optional, skip step if false
  on?: {                                  // goto branching
    success?: { goto: string };
    failure?: { goto: string };
  };
  env?: Record<string, string>;
  secrets?: string[];                     // env var names to expose, masked in logs
  output_capture?: OutputCapture;         // default: text
  allow_parse_error?: boolean;            // for json capture
  depends_on?: DependsOnConfigBasic;
  wait_for?: WaitForConfig;               // mutually exclusive with command/provider
}

interface StepCommand extends StepBase {
  command: string[];                      // e.g., ["echo","hello"]
}

interface StepProvider extends StepBase {
  provider: string;                       // e.g., 'claude'
  provider_params?: ProviderParams;       // e.g., { model: "..." }
  command_override?: string[];            // replaces provider template
  input_file?: string;                    // prompt path (read into PROMPT)
  output_file?: string;                   // redirect stdout to file
}

interface StepForEach extends StepBase {
  for_each: ForEachBlock;
}

type Step = StepCommand | StepProvider | StepForEach | StepBase; // StepBase used only for wait_for-only steps

interface WorkflowSpec {
  version: string;                        // '1.1'
  name?: string;
  strict_flow?: boolean;                  // sequential default; goto allowed
  providers?: Record<string, ProviderTemplate>;
  inbox_dir?: string;                     // defaults per spec
  processed_dir?: string;
  failed_dir?: string;
  task_extension?: string;                // default: '.task'
  steps: Step[];
  context?: Record<string, string>;       // initial context vars
}
```

### 5.2 Run state (authoritative)

```ts
interface StepDebugInfo {
  command?: string[];
  cwd?: string;
  env_count?: number;
}

interface StepErrorContext {
  message: string;
  exit_code: number;
  stdout_tail?: string[];
  stderr_tail?: string[];
  context?: {
    undefined_vars?: string[];
    failed_deps?: string[];
    substituted_command?: string[];
  };
}

interface StepResultBase {
  status: 'pending' | 'running' | 'completed' | 'failed' | 'skipped';
  exit_code?: number;
  started_at?: string;
  completed_at?: string;
  duration_ms?: number;
  truncated?: boolean;
  debug?: StepDebugInfo;
  error?: StepErrorContext;
}

interface StepResultText extends StepResultBase { output?: string; }
interface StepResultLines extends StepResultBase { lines?: string[]; }
interface StepResultJson extends StepResultBase { json?: any; }
type StepResult = StepResultText | StepResultLines | StepResultJson;

interface ForEachRunState {
  items: string[];
  completed_indices: number[];
  current_index?: number;
}

interface RunState {
  schema_version: '1.1.1';
  run_id: string;                          // e.g., 20250115T143022Z-a3f8c2
  workflow_file: string;
  workflow_checksum: string;               // sha256
  started_at: string;
  updated_at: string;
  status: 'running' | 'completed' | 'failed' | 'suspended';
  context: Record<string, string>;
  steps: Record<string, StepResult>;
  for_each?: Record<string, ForEachRunState>;
}
```

### 5.3 Status JSON

```ts
interface StatusJsonV1 {
  schema: 'status/v1';
  correlation_id?: string;
  agent?: string;
  run_id?: string;
  step?: string;
  timestamp: string;
  success: boolean;
  exit_code: number;
  outputs?: string[];          // relative to WORKSPACE
  metrics?: Record<string, number>;
  next_actions?: { agent: string; file: string }[];
  message?: string;
}
```


## 6) Execution Flow (pseudocode)

1. CLI parses command and options; resolves `WORKSPACE` (cwd) and `RUN_ROOT`.
2. Load YAML workflow → validate minimal structure → compute checksum.
3. Create run_id and initialize state.json; or load/validate existing on `resume`.
4. For each top-level step index i:
   - Evaluate `when.equals` if present. If false, mark skipped as completed (exit_code 0) and continue.
     - Default semantics: mark step `status: 'skipped'` when the condition is false (no process execution).
   - If `wait_for` present: run wait loop, record metrics and result, handle timeout (exit 124) and `on.failure`.
   - If `for_each` present: resolve items via pointer or literal; persist into state; iterate items:
     - Set loop variables (`item`, `loop.index`, `loop.total`).
     - Execute nested steps recursively (same rules), tracking per-iteration progress in state.
   - Else (command/provider step):
     - Resolve dependencies (globs) with variable substitution; error out (exit 2) if required missing.
     - Build `PROMPT` from `input_file` contents, applying dependency injection if requested.
     - Build argv from provider template + `provider_params`, or from `command_override`, or raw `command`.
     - Spawn process; capture stdout/stderr per `output_capture` mode; enforce limits.
     - Redirect stdout to `output_file` if specified; still capture for state (truncated rules apply).
     - Record timings, exit_code; apply retries if configured; update state atomically after each attempt.
   - Apply branching (`on.success`/`on.failure` goto). If goto taken, set i accordingly; else proceed sequentially.
   - Control flow defaults:
     - `strict_flow` default true: any non-zero exit halts the run unless an `on.failure.goto` is defined for that step.
     - `_end` is a reserved target that terminates the run successfully.
5. On success, optionally `--archive-processed` zip processed/ to `RUN_ROOT/processed.zip` (or provided path).
6. Update global run status to completed/failed; finalize logs.


## 7) Variable Model & Substitution

- Namespaces (precedence): run scope → loop scope → step results → context.
- Substitution locations: argv arrays, file paths, provider parameters, conditional values, dependency globs.
- No substitution inside file contents; to include dynamic content, preprocess in a step and reference that file.
- Implementation-defined choices (defaults for this implementation):
  - Undefined variables: error and halt with exit code 2. Flag `--undefined-as-empty` downgrades to empty string with warning.
  - Type coercion in conditions: compare as strings; JSON numbers are stringified.
  - Escape syntax: treat `$$` as a single literal `$`. To render `${`, write `$${`.
  - Disallow `${env.*}` namespace in workflows; loader rejects such references.


## 8) Provider Integration

- Provider registry holds templates:
  - Example (Claude): `command: ["claude","-p","${PROMPT}","--model","${model}"]`
  - `defaults.model = 'claude-sonnet-4-20250514'` (configurable)
- Command construction precedence:
  1) `command_override` (step) if present
  2) Provider template + merged params (`defaults` < `provider_params`)
  3) Raw `command` (no provider) if specified
- Input handling: if `input_file`, read contents to PROMPT after optional dependency injection; pass as CLI argument (not stdin).
- Output handling: if `output_file`, redirect child stdout to that file while still capturing for state until size limits.
- Exit codes: 0 success; 1 retryable API error; 2 invalid input/non-retryable; 124 timeout (retryable).

Contracts (summary):
- Command construction precedence is enforced; template interpolation errors surface as `MissingTemplateKeyError` and map to exit code 2 at the step level.
- Prompt handling is in-memory: `input_file` contents are read, optional dependency injection applies, then the composed prompt is passed via argv as `${PROMPT}`.


## 9) Dependency Validation & Injection

- Resolution
  - Evaluate `required` and `optional` globs after variable substitution against WORKSPACE.
  - required + 0 matches → exit code 2; optional + 0 matches → continue.
  - Directories count as existing; follow symlinks; re-evaluate each loop iteration.

- Injection
  - Basic: `inject: true` → default list mode + default instruction prepended.
  - Advanced: `inject: { mode, instruction, position }`.
  - list mode: prepend/append instruction then bullet list of matched files (relative paths).
  - content mode: include file contents; recommended fence format:
    ```
    <instruction>
    --- BEGIN REQUIRED FILES ---
    [path]
    ```
    <file contents>
    ```
    ... (repeat)
    --- END REQUIRED FILES ---
    ```
  - No modification to source `input_file`; the composed prompt string is passed to provider argv.

Contracts (summary):
- Validate required and optional patterns after variable substitution; use POSIX globs (`*` and `?`). Do not support `**` (globstar).
- Missing required → non-retryable validation error mapped to exit code 2; optional missing → proceed without warning unless `--verbose`.
- Injection modifies only the in-memory prompt composition; never edits files on disk.


## 10) Wait / Queue Semantics

- `wait_for` steps
  - Poll `glob` every `poll_ms` until `min_count` matches or `timeout_sec` elapses.
  - On timeout, exit 124; record `files`, `wait_duration`, `poll_count` in state.

- Queue operations
  - Write task: create `*.tmp` then atomic rename to `*.task`.
  - On success: `mv <task> processed/{timestamp}/` (create dir as needed)
  - On non-retryable failure: `mv <task> failed/{timestamp}/`

Inter‑Agent Example (worked):
- Architect step prompts provider to write `artifacts/architect/system_design.md`.
- A subsequent bash step lists architect artifacts and writes `inbox/engineer/task_X.tmp` → rename to `.task`.
- Engineer step requires `artifacts/architect/system_design.md`; with `depends_on.inject: true`, the orchestrator prepends a file list to the in-memory prompt before invoking the provider.
- Engineer produces code; orchestrator writes a QA review task to `inbox/qa/` and optionally waits for feedback with `wait_for` on `inbox/engineer/*.task`.


## 11) Observability & Logs

- Logs under `RUN_ROOT/logs/`:
  - `orchestrator.log` main log
  - `StepName.stdout` (>8KB or JSON parse error)
  - `StepName.stderr` (if non-empty)
  - `StepName.debug` (when `--debug`)
- Progress UI (when `--progress`): `[n/N] StepName: Running (Xs)...` with for-each progress `[i/total]`.
- JSON output mode (`--json`): stream machine-readable updates to stdout (optional enhancement).
- Secrets masking: redact values of `secrets` env names in logs and command echoes, including real-time masking for streamed output.


## 12) Error Handling & Retry

- Per-step outcome → apply `on.success`/`on.failure` goto if present; otherwise continue sequentially.
- CLI `--on-error stop|continue|interactive` governs default behavior when not overridden.
- Retries: `--max-retries`, `--retry-delay` apply to retryable exit codes (1, 124).
- On failure, record error context (message, exit_code, tails, substituted command, failed deps, undefined vars).


## 13) State Integrity, Resume, and Cleanup

- Atomic state writes: write `state.json.tmp` then `rename`.
- Before each step, if `--backup-state` (or always per spec recommendation): copy `state.json` → `state.json.step_<step>.bak` (keep last 3).
- On corruption: `resume --repair` attempts last valid backup; `resume --force-restart` starts new run (new run_id); always validate checksum.
- `clean --older-than 7d` removes old run directories; `--state-dir` can override default base path.

Contracts (summary):
- Persist `state.json` after every step attempt (success or failure) to enable deterministic resume.
- On parse or write failure, do not partially overwrite: use tmp + atomic rename.


## 14) CLI Contract & Safety Rails

- Run
  - `orchestrate run <workflow.yaml> --context k=v --context-file ctx.json [--clean-processed] [--archive-processed <dst>]`
  - `--clean-processed` only operates on `WORKSPACE/processed/`; refuse others.
  - `--archive-processed` must target path outside `processed/`; default to `RUN_ROOT/processed.zip`.

- Resume / Run-step / Watch
  - `orchestrate resume <run_id>` validates state, resumes.
  - `orchestrate run-step <name> --workflow <file>` executes a single step in isolation (advanced usage).
  - `orchestrate watch <workflow.yaml>` (optional): re-run on file changes.


## 15) Implementation Details & Libraries

- Parsing & schema: `js-yaml`, minimal shape checks; enforce pointer grammar strings.
- FS ops: `fs-extra` (atomic writes via tmp+rename), `glob` for POSIX patterns with symlink following.
- CLI: `commander` for commands/flags; `chalk` for color; `ora` for progress.
- Zipping: `archiver` or manual `zip` spawn (choose minimal dependency).
- Time/IDs: use UTC ISO strings; run_id format `YYYYMMDDTHHMMSSZ-<6char>`.


## 16) Testing Strategy (maps to Acceptance Tests)

- Unit tests (Jest)
  - substitution: namespaces, precedence, undefined handling
  - pointers: `steps.X.lines`, `steps.X.json.path`
  - depends_on: required/optional, globs, loop re-eval, variable substitution
  - injector: list/content modes, prepend/append, optional omissions
  - output capture: text/lines/json, limits, truncation, oversize JSON (exit 2), `allow_parse_error`
  - wait_for: timeout code 124, metrics recorded
  - provider argv: template + defaults + overrides, `command_override`
  - CLI safety: clean/archive constraints

- Integration tests
  - end-to-end sample workflow mirroring spec example
  - inbox atomicity (.tmp → rename), processed/failed moves
  - resume after simulated crash; repair from backup


## 17) Step-by-step Build Plan (for implementation)

1) Scaffolding
  - CLI command skeletons; logging; basic config defaults.

2) Workflow loader
  - YAML load; minimal validation; checksum.

3) State manager
  - Run ID, RUN_ROOT, atomic writes, backups, resume/repair.

4) Substitution & pointers
  - Implement namespace precedence; `${}` scanning; `$$` escape; pointer resolution.

5) Dependency resolver
  - Globs, required/optional semantics; symlink behavior; error mapping.

6) Injector
  - list/content modes; prepend/append; compose prompt string.

7) Step executor
  - Provider registry, argv builder, command_override; env/secrets; spawn; capture modes; truncation; retries.

8) Control flow
  - when.equals; on.success/on.failure goto; strict sequential default.

9) Wait/Queue
  - `wait_for` polling and metrics; atomic queue ops; processed/failed moves.

10) Observability
  - Logs, progress output; JSON mode; error context tails.

11) CLI safety & utilities
  - clean/archive; state-dir override; environment variables mapping.

12) Tests
  - Implement unit + integration tests mapped to acceptance list.


## 18) Implementation-defined Defaults (explicit choices)

- Undefined variable policy: error (exit 2). Optional `--undefined-as-empty` flag replaces with empty string and logs a warning once per var.
- Condition comparisons: strict string comparison; both sides coerced to strings.
- Escape syntax: `$$` → `$`, `$${` → `${`.
- Output capture limits: as in spec (text: 8KB in state; spill >1MB to file; lines: 10,000 max; json: parse up to 1MB unless `allow_parse_error`).
- Retries: default 0; classify 1 and 124 as retryable.


## 19) Security & Safety

- Secrets handling: only pass env vars named in `secrets`; never print values; mask in logs (including streamed output).
- Path safety: enforce ADR-01 rules strictly—reject absolute or `..` paths; follow symlinks but reject if resolution escapes WORKSPACE; validate at load time and before FS ops.
- Clean/archive safeguards strictly enforced per spec.


## 20) Developer Notes

- Keep step names unique (keys in state map).
- Avoid modifying input files during injection; always compose in-memory prompt string.
- Ensure state updates are granular: write after each step attempt to support robust resumes.
- Preserve compatibility with future v2 caching by being explicit about dependencies in state for each step.
- Testability: design for dependency injection—wire Engine with pluggable StateManager, StepExecutor, and Runner to ease unit testing and mocking.


## 21) Appendix — Main Run Loop Sketch

```ts
async function run(workflowPath: string, opts: CliOptions) {
  const wf = await loadWorkflow(workflowPath);
  const run = await createOrLoadRunState(wf, opts);
  let i = 0;
  while (i < wf.steps.length) {
    const step = wf.steps[i];
    backupState(run, step.name, opts);
    const outcome = await executeStep(step, run, wf, opts);
    await writeState(run);
    const next = nextIndexFromOutcome(outcome, wf, i);
    if (next === 'END') break;
    i = next ?? (i + 1);
  }
  finalizeRun(run);
}
```

This architecture maps 1:1 with the v1.1 spec and provides enough structure for a junior engineer to start implementing modules in order, with clear interfaces and well-defined behaviors.
