# Architecture: LLM Workflow Orchestrator (Spec-Complete)

**Status:** Accepted

This document outlines the architecture for the LLM Workflow Orchestrator. It details the core design principles, system components, data flows, and the explicit, spec-complete IDL contracts necessary to guide its implementation. This version has been refined to close specific gaps between the high-level design and the source specification.

## 1. Context and Goals

Modern LLM-based agent systems require a robust mechanism to execute sequences of tasks, manage dependencies between them, and coordinate their interactions. The primary goals of this architecture are:

*   **Determinism & Reproducibility:** Workflows must execute in a predictable, sequential order.
*   **Observability & Debuggability:** The state of the system, including inter-agent communication, must be transparent and easy to inspect.
*   **Extensibility:** The system must easily integrate with various command-line tools (LLM providers, build tools, etc.) without modification to the core engine.
*   **Robustness:** The system must be able to recover from interruptions and resume failed workflows.
*   **Developer Experience:** Workflows should be defined in a simple, human-readable, and version-controllable format.

## 2. Core Architectural Principles & Decisions

### ADR-01: Filesystem as the Source of Truth

**Decision:** The filesystem is the primary medium for state, artifacts, and inter-agent communication. We will use a structured directory layout (`inbox/`, `artifacts/`, `processed/`, etc.) instead of a database, message queue, or in-memory data store.

**Rationale:**
*   **Transparency:** Any developer can inspect the state of the system using standard tools (`ls`, `cat`, `find`). This drastically simplifies debugging.
*   **Simplicity:** It removes the need for additional infrastructure (databases, brokers), making the system lightweight and easy to set up.
*   **Durability:** The state is inherently persistent, providing a robust foundation for resuming workflows.
*   **Decoupling:** Agents communicate by reading and writing files, a universal and language-agnostic mechanism. An agent doesn't need to know about the orchestrator, only its workspace.

### ADR-02: Declarative YAML for Workflow Definition

**Decision:** Workflows will be defined in YAML files. The orchestrator will interpret these files to drive execution.

**Rationale:**
*   **Readability:** YAML is human-friendly and well-suited for defining structured data like a workflow.
*   **Version Control:** Workflow logic can be checked into Git, reviewed, and versioned alongside source code and prompts.
*   **Separation of Concerns:** It separates the *what* (the workflow logic) from the *how* (the orchestration engine's implementation).

### ADR-03: Sequential, Single-Threaded Execution Model

**Decision:** The orchestrator will execute one step at a time in a single thread. Concurrency and parallel execution are explicitly out of scope for the initial version.

**Rationale:**
*   **Determinism:** This is the simplest model to guarantee a predictable and reproducible execution order, which is critical for debugging complex agent interactions.
*   **Simplicity of Implementation:** It avoids the complexities of managing concurrent state, race conditions, and deadlocks.

### ADR-04: Loose Coupling via CLI Provider Contract

**Decision:** The orchestrator will interact with external tools (like `claude`, `gemini`, or `bash`) as "black box" command-line processes. It will not use direct API or library integrations.

**Rationale:**
*   **Extensibility:** Any tool that can be invoked from a command line can be integrated as a provider without changing the orchestrator's code.
*   **Isolation:** The orchestrator is isolated from the internal complexities and dependencies of the tools it runs.
*   **Refined Contract ("Managed Black Box with Assumed Side Effects"):**
    1.  **Explicit Contract:** The orchestrator directly manages command-line invocation, `stdout`/`stderr`/`exit_code` capture, and environment variable injection. Prompts are passed as a single CLI argument (argv), not via STDIN.
    2.  **Implicit Contract:** The orchestrator *assumes* providers can read/write files as side effects. It validates these side effects using `depends_on` checks and subsequent steps.

### ADR-05: Explicit JSON State for Resumability

**Decision:** The complete state of each run will be persisted to a `state.json` file. This file will be updated after every step.

**Rationale:**
*   **Robustness:** If the orchestrator process is terminated, it can be resumed by loading this state file and determining the last successfully completed step.
*   **Auditability:** The state file serves as a detailed, machine-readable log of the entire workflow execution.

## 3. System Component IDL Contracts

This section defines the strict Interface Definition Language (IDL) for each system component. Implementations **must** adhere precisely to these contracts.

---
### **File: `orchestrator/models/workflow_IDL.md`**

```idl
module orchestrator.models.workflow {

    // --- Typed structs for control flow ---
    struct EqualsCondition {
        left: string;
        right: string;
    }

    struct WhenClause {
        equals: EqualsCondition;
    }

    struct GotoAction {
        goto: string;
    }

    struct OnHandlers {
        success: optional GotoAction;
        failure: optional GotoAction;
    }
    
    // --- Typed struct for injection config ---
    struct InjectConfig {
        mode: string; // "list" | "content" | "none"
        instruction: optional string;
        position: string; // "prepend" | "append"
    }

    struct DependsOn {
        required: optional list<string>;
        optional: optional list<string>;
        inject: union<boolean, InjectConfig>;
    }

    struct WaitFor {
        glob: string;
        timeout_sec: int;
        poll_ms: int;
        min_count: int;
    }

    struct ProviderConfig {
        command: list<string>;
        defaults: optional dict<string, Any>;
    }

    // --- Step struct with spec-aligned fields ---
    struct Step {
        name: string;
        agent: optional string;
        output_capture: string; // "text" | "lines" | "json"
        for_each: optional dict<string, Any>;
        allow_parse_error: optional boolean;
        provider: optional string;
        provider_params: optional dict<string, Any>;
        command_override: optional list<string>;
        wait_for: optional WaitFor;
        depends_on: optional DependsOn;
        on: optional OnHandlers;
        when: optional WhenClause;
        env: optional dict<string, string>;
        secrets: optional list<string>;
        // Spec-aligned IO:
        // - input_file: When set with a provider, the orchestrator reads the file contents
        //   and passes the prompt text as a single CLI argument to the provider.
        // - output_file: When set, the orchestrator redirects provider STDOUT to this file
        //   while also applying the configured output_capture for state.
        input_file: optional string;
        output_file: optional string;
    }

    struct Workflow {
        version: string;
        name: string;
        strict_flow: optional boolean;
        providers: optional dict<string, ProviderConfig>;
        context: optional dict<string, Any>;
        steps: list<Step>;
    }
}
```
---
### **File: `orchestrator/models/state_IDL.md`**
```idl
# @depends_on(orchestrator.models.workflow)
module orchestrator.models.state {

    struct WaitState {
        files: list<string>;
        wait_duration: int; // milliseconds
        poll_count: int;
    }

    struct StepResult {
        step_name: string;
        status: string; // "succeeded" | "failed" | "skipped"
        exit_code: int;
        start_time: string; // ISO 8601 format
        end_time: string;   // ISO 8601 format
        duration: float;    // in seconds
        // Note: when output_capture is 'lines' or 'json', omit 'output' in state.
        output: optional string;
        lines: optional list<string>;
        json: optional Any;
        truncated: boolean;
        parse_error: optional boolean; // for json mode with allow_parse_error
        stdout_log: optional string;   // spill/log path under RUN_ROOT/logs
        wait: optional WaitState;      // populated when step used wait_for
    }

    struct RunState {
        run_id: string;
        workflow_name: string;
        status: string; // "running" | "succeeded" | "failed"
        start_timestamp: string; // ISO 8601 format
        end_timestamp: optional string;
        variables: dict<string, Any>;
        step_results: dict<string, StepResult>;
    }
}
```
---
### **File: `orchestrator/parsing/loader_IDL.md`**
```idl
# @depends_on(orchestrator.models.workflow.Workflow)
# @depends_on_resource(type="FileSystem", purpose="Reading the workflow YAML file")
module orchestrator.parsing.loader {

    interface LoaderFunctions {
        // preconditions:
        // - The file at `path` must exist and be a valid YAML file.
        // - The YAML content must conform to the structure of the Workflow struct.
        // postconditions:
        // - Returns a fully validated Workflow data structure.
        // behavior:
        // - Reads a YAML file from the filesystem.
        // - Deserializes the YAML content.
        // - Validates the deserialized data against the Workflow data contract.
        // - Validates that disallowed namespaces (e.g., ${env.*}) are not present.
        // - Validates pointer grammar for `for_each.items_from` to ensure it targets an array.
        // @raises_error(condition="FileNotFound", description="Raised if the workflow file does not exist.")
        // @raises_error(condition="YamlParseError", description="Raised if the file content is not valid YAML.")
        // @raises_error(condition="WorkflowValidationError", description="Raised if YAML does not match the required structure.")
        orchestrator.models.workflow.Workflow load_workflow(string path);
    }
}
```
---
### **File: `orchestrator/context/variables_IDL.md`**
```idl
# @depends_on(orchestrator.models.state.RunState)
module orchestrator.context.variables {

    interface VariableManager {
        // preconditions:
        // - state must be a valid, initialized RunState object.
        // postconditions:
        // - Returns an initialized VariableManager ready for substitutions.
        // behavior:
        // - Factory method to create an instance tied to a specific run state.
        VariableManager create_from_state(orchestrator.models.state.RunState state);

        // preconditions:
        // - The instance has been initialized with a RunState.
        // - `text` is a string that may contain `${...}` placeholders.
        // postconditions:
        // - Returns a new string with all valid placeholders substituted with their values.
        // behavior:
        // - Resolves variables by searching namespaces in a fixed precedence order.
        //
        // Supported Namespaces & Precedence:
        // 1. `run`: Global variables for the current run.
        //    - `${run.timestamp_utc}`: The UTC start time of the run (YYYYMMDDTHHMMSSZ).
        // 2. `loop`: Variables available only inside a `for_each` block.
        //    - `${item}`: The current item in the iteration.
        //    - `${loop.index}`: The 0-based index of the current iteration.
        //    - `${loop.total}`: The total number of items in the iteration.
        // 3. `steps`: Results from previously executed steps.
        //    - `${steps.<StepName>.exit_code}`
        //    - `${steps.<StepName>.output}` (text mode)
        //    - `${steps.<StepName>.lines}` (lines mode, returns an array)
        //    - `${steps.<StepName>.json}` (json mode, returns an object)
        //    - `${steps.<StepName>.duration}`
        // 4. `context`: User-defined variables from the workflow's `context` block.
        //    - `${context.<key>}`
        //
        // Pointer Grammar:
        // - For `steps.<StepName>.json`, dot-notation is supported to access nested values (e.g., `${steps.MyStep.json.result.files}`) that resolve to arrays when used in `for_each.items_from`.
        // - Wildcards, array indexing, or advanced expressions are NOT supported in pointers.
        //
        // Error Conditions:
        // - The `${env.*}` namespace is explicitly disallowed and will raise an error.
        // - Unresolved placeholders raise UndefinedVariableError and halt execution (fail-fast).
        //
        // @raises_error(condition="UndefinedVariableError", description="Raised if a placeholder cannot be resolved in any namespace. Execution halts.")
        string substitute(string text, optional Any loop_item, optional int loop_index);
    }
}
```
---
### **File: `orchestrator/execution/runner_IDL.md`**
```idl
# @depends_on_resource(type="ProcessExecutor", purpose="Running external command-line processes")
module orchestrator.execution.runner {
    
    struct CommandResult {
        exit_code: int;
        stdout: string;
        stderr: string;
    }

    interface CommandRunner {
        // preconditions:
        // - `command` is a list of strings representing the command and its arguments.
        // - `env` is a dictionary of environment variables to set for the process.
        // postconditions:
        // - The external command has completed execution.
        // - Returns a CommandResult struct containing the exit code and captured output streams.
        // behavior:
        // - Spawns a new system process.
        // - Sets the specified environment variables.
        // - Captures stdout and stderr streams.
        // - Masks any values from `secrets_to_mask` in the real-time logged output.
        // - Enforces the specified timeout; on timeout, terminates the process and returns exit_code 124 (retryable).
        CommandResult run_command(list<string> command, dict<string, string> env, list<string> secrets_to_mask, int timeout_sec);
    }
}
```
---
### **File: `orchestrator/execution/output_IDL.md`**
```idl
module orchestrator.execution.output {
    
    struct ProcessedOutput {
        output: optional string;      // text mode: first 8 KiB stored; >1 MiB spills to log
        lines: optional list<string>; // lines mode: max 10,000 lines
        json: optional Any;           // json mode: parsed object when within 1 MiB
        truncated: boolean;           // true when spill/truncation occurred
        parse_error: optional boolean; // json mode with allow_parse_error=true
        stdout_log: optional string;   // path to spilled stdout under RUN_ROOT/logs
    }

    interface OutputProcessor {
        // preconditions:
        // - `stdout` is the raw string output from a command.
        // postconditions:
        // - Returns a ProcessedOutput struct populated according to the specified mode.
        // behavior:
        // - 'text': Store up to 8 KiB of stdout in state. If total stdout > 1 MiB, spill full stdout to RUN_ROOT/logs and set truncated=true and stdout_log.
        // - 'lines': Split into lines, store first 10,000 entries; if more, set truncated=true.
        // - 'json': Buffer up to 1 MiB; parse as JSON. If buffer exceeds limit or parsing fails:
        //     * if allow_parse_error=false: raise JsonParseError (caller maps to exit code 2).
        //     * if allow_parse_error=true: set parse_error=true, json omitted, and spill raw stdout to RUN_ROOT/logs.
        // Note: when mode is 'lines' or 'json', callers must omit 'output' from StepResult to avoid duplication.
        // @raises_error(condition="JsonParseError", description="Raised if mode is 'json' and parsing fails with allow_parse_error=false.")
        ProcessedOutput process(string stdout, string mode, boolean allow_parse_error);
    }
}
```
---
### **File: `orchestrator/execution/dependencies_IDL.md`**
```idl
# @depends_on(orchestrator.models.workflow.DependsOn)
# @depends_on(orchestrator.context.variables.VariableManager)
# @depends_on_resource(type="FileSystem", purpose="Validating file existence and reading content for injection")
module orchestrator.execution.dependencies {
    
    interface DependencyManager {
        // preconditions:
        // - `depends_on` is a valid DependsOn data structure from a workflow step.
        // - `var_manager` is an initialized VariableManager.
        // - `prompt_content` is the original prompt string before injection.
        // postconditions:
        // - If validation succeeds, returns the prompt content, potentially modified by injection.
        // - If validation fails, an error is raised.
        // behavior:
        // - First, resolves any variables in the `required` and `optional` path patterns.
        // - Pattern syntax supports '*', '?', and '**' (globstar) for recursive matching.
        // - Checks for the existence of all files matching the resolved `required` patterns; zero matches produce a validation error.
        // - If `inject` is enabled, it resolves file paths/content and prepends or appends it to `prompt_content` according to the injection mode. Injection modifies the in-memory prompt only; input files on disk are unchanged.
        // @raises_error(condition="DependencyNotFoundError", description="Raised if a 'required' file pattern matches no existing files.")
        string validate_and_inject(orchestrator.models.workflow.DependsOn depends_on, orchestrator.context.variables.VariableManager var_manager, string prompt_content);
    }
}
```
---
### **File: `orchestrator/execution/executor_IDL.md`**
```idl
# @depends_on(orchestrator.models.workflow.Step)
# @depends_on(orchestrator.models.state.StepResult)
# @depends_on(orchestrator.context.variables.VariableManager)
# @depends_on(orchestrator.execution.runner.CommandRunner)
# @depends_on(orchestrator.execution.output.OutputProcessor)
# @depends_on(orchestrator.execution.dependencies.DependencyManager)
module orchestrator.execution.executor {

    interface StepExecutor {
        // preconditions:
        // - `step` is a valid Step object from the workflow.
        // - `var_manager` is an initialized VariableManager for the current run state.
        // postconditions:
        // - The step has been fully executed.
        // - Returns a StepResult object detailing the outcome.
        // behavior:
        // - Orchestrates the entire lifecycle of a single step's execution:
        //   1. Handles the `wait_for` primitive, blocking if necessary.
        //   2. Resolves variables within the Step definition.
        //   3. Calls the DependencyManager to validate dependencies and perform prompt injection, producing the final prompt content.
        //   4. Constructs the final command to be executed (see "Command Construction Logic" below).
        //   5. Calls the CommandRunner to execute the process.
        //   6. Calls the OutputProcessor to handle the command's stdout.
        //   7. Assembles and returns the final StepResult.
        //
        // Command Construction Logic:
        // - If `command_override` is present, it is used directly and all provider logic is skipped.
        // - Otherwise, the command is constructed from the step's `provider` definition:
        //   1. The base `command` array from the `providers:` block is used as a template.
        //   2. The template is interpolated using a map of parameters with the following precedence:
        //      a. Step-level `provider_params` (highest precedence).
        //      b. Provider-level `defaults`.
        //   3. The orchestrator provides `${PROMPT}` as the final prompt content (after dependency injection). Prompts are passed as a single CLI argument (argv).
        //   4. If `input_file` is set, its literal contents become the initial prompt content before dependency injection.
        //   5. If `output_file` is set, STDOUT is redirected to that file during execution (OutputProcessor also applies capture limits for state).
        //
        // @raises_error(condition="MissingTemplateKeyError", description="Raised if the provider command template references a key not found in provider_params or defaults.")
        orchestrator.models.state.StepResult execute_step(orchestrator.models.workflow.Step step, orchestrator.context.variables.VariableManager var_manager);
    }
}
```
---
### **File: `orchestrator/core/state_IDL.md`**
```idl
# @depends_on(orchestrator.models.state.RunState)
# @depends_on(orchestrator.models.workflow.Workflow)
# @depends_on_resource(type="FileSystem", purpose="Reading and writing run state files")
module orchestrator.core.state {

    interface StateManager {
        // preconditions:
        // - If run_id is provided, a corresponding state file might exist.
        // - `workflow` is a valid, loaded Workflow object.
        // - `initial_context` is a dictionary of user-provided startup variables.
        // postconditions:
        // - If resuming, returns the existing RunState loaded from the filesystem.
        // - If starting a new run, returns a new RunState object, initialized with run_id, workflow details, and initial context.
        // behavior:
        // - RUN_ROOT is `.orchestrate/runs/${run_id}` under WORKSPACE.
        // - For a resume operation (`run_id` is not null), it loads the state from `${RUN_ROOT}/state.json`.
        // - For a new run, it creates a new RunState instance, with a new run_id and the provided workflow and context data.
        // @raises_error(condition="StateNotFound", description="Raised on resume if the state file for run_id does not exist.")
        // @raises_error(condition="StateCorrupted", description="Raised if the state file is invalid JSON or fails validation.")
        orchestrator.models.state.RunState load_or_create(optional string run_id, orchestrator.models.workflow.Workflow workflow, dict<string, Any> initial_context);

        // preconditions:
        // - `state` must be a valid RunState object.
        // postconditions:
        // - The provided RunState is serialized to JSON and saved to `${RUN_ROOT}/state.json`.
        // behavior:
        // - Persists the current state of the workflow run to the filesystem, enabling resume functionality.
        // @raises_error(condition="IOError", description="Raised if the state file cannot be written.")
        void save(orchestrator.models.state.RunState state);
    }
}
```
---
### **File: `orchestrator/core/engine_IDL.md`**
```idl
# @depends_on(orchestrator.models.workflow.Workflow)
# @depends_on(orchestrator.core.state.StateManager)
# @depends_on(orchestrator.execution.executor.StepExecutor)
module orchestrator.core.engine {

    interface Orchestrator {
        // preconditions:
        // - `workflow` is a valid, loaded Workflow object.
        // postconditions:
        // - An Orchestrator instance is created and ready to execute the workflow.
        // behavior:
        // - Constructor for the main engine. It is initialized with the static workflow definition and its dependencies (like the StateManager and StepExecutor implementations).
        Orchestrator create(orchestrator.models.workflow.Workflow workflow, orchestrator.core.state.StateManager state_manager, orchestrator.execution.executor.StepExecutor step_executor);

        // preconditions:
        // - The Orchestrator has been initialized.
        // - `initial_context` is a map of user-provided variables.
        // - `run_id` is null for a new run, or a valid ID for a resume.
        // postconditions:
        // - The workflow has executed to completion or has failed.
        // - The final run status is returned (e.g., "succeeded" or "failed").
        // behavior:
        // - The main control loop of the application.
        //   1. Initializes or loads the RunState.
        //   2. Determines the starting step (step 0 for new runs, the next un-executed step for resumes).
        //   3. Iteratively executes steps using the StepExecutor in listed order unless redirected by `goto`.
        //   4. Handles control flow logic:
        //      - Default progression: next listed step.
        //      - `_end` is a reserved target that terminates the run successfully.
        //      - `on.success.goto` when exit_code == 0; `on.failure.goto` when exit_code != 0 (including dependency/parse/timeouts).
        //      - strict_flow (default true): any non-zero exit halts the run unless an `on.failure.goto` is defined for that step.
        //      - `when`: if false, mark step as "skipped" without executing.
        //      - `for_each`: resolve items once at start, persist to state for deterministic resume; execute indices in order.
        //   5. Updates the RunState after each step.
        //   6. Returns the final status of the run.
        string run(dict<string, Any> initial_context, optional string run_id);
    }
}
```
---
### **File: `orchestrator/cli_IDL.md`**
```idl
# @depends_on(orchestrator.core.engine.Orchestrator)
module orchestrator.cli {

    interface CliFunctions {
        // preconditions:
        // - Command-line arguments have been parsed.
        // postconditions:
        // - The orchestrator has been invoked to run a workflow.
        // - The application exits with code 0 on success, non-zero on failure.
        // behavior:
        // - The main entry point of the application.
        // - Supports commands:
        //   * `orchestrate run <workflow.yaml> [--context k=v ...] [--context-file file] [--clean-processed] [--archive-processed [dest.zip]]`
        //   * `orchestrate resume <run_id>`
        //   * `orchestrate run-step <step_name> --workflow <workflow.yaml>`
        //   * `orchestrate watch <workflow.yaml>`
        // - Safety:
        //   * `--clean-processed` only operates on WORKSPACE/processed and refuses other paths.
        //   * `--archive-processed` destination must be outside processed/; default to `${RUN_ROOT}/processed.zip` if omitted.
        // - Bootstraps Loader, StateManager, Engine, and invokes the appropriate method.
        // - Handles top-level exceptions and translates them to user-friendly error messages and appropriate exit codes.
        void main();
    }
}
```

## 4. Key Data Flows and Interactions

### A. Standard Workflow Execution (`orchestrate run`)

1.  **CLI:** The `cli` module parses the command (`run`), workflow path, and initial context variables.
2.  **Engine Init:** It instantiates the `OrchestrationEngine`.
3.  **Loading:** The `Engine` calls the `Loader` to parse the YAML file into a `Workflow` model.
4.  **State Init:** The `Engine` calls the `StateManager` to create a new `RunState` object, populating it with a new `run_id` and the initial context.
5.  **Execution Loop:** The `Engine` enters its main loop:
    a. It determines the next `Step` to execute.
    b. It instantiates the `StepExecutor` with the current `Step` and `VariableManager`.
    c. **Execution:** The `StepExecutor` performs its sequence:
        i.  Validates dependencies via the `DependencyManager`.
        ii. Builds the final command, potentially modifying the prompt via the `DependencyManager`'s injection logic.
        iii. Executes the command via the `CommandRunner`.
        iv. Processes the result via the `OutputProcessor`.
        v.  Returns a `StepResult` model.
    d. **State Update:** The `Engine` receives the `StepResult`, updates the `RunState` object, and calls the `StateManager` to persist the new state to `state.json`.
    e. The loop continues until the workflow ends or an unhandled error occurs.

### B. Inter-Agent Communication (via Filesystem)

This flow illustrates how two agents, "Architect" and "Engineer," coordinate.

1.  **Architect Step:**
    a. The `StepExecutor` runs a `claude` command for the "Architect" agent.
    b. The prompt instructs the agent to write its output to `artifacts/architect/system_design.md`.
    c. The `claude` process runs and creates this file as a side effect. The step completes successfully.
2.  **Task Creation Step:**
    a. A subsequent `bash` step in the workflow runs.
    b. It lists the files in `artifacts/architect/` and writes their paths into a new task file: `inbox/engineer/task_123.tmp`.
    c. The script then performs an `mv` (atomic rename) to `inbox/engineer/task_123.task`.
3.  **Engineer Step:**
    a. A later step for the "Engineer" agent begins.
    b. Its `depends_on` block requires `artifacts/architect/system_design.md` to exist. The `DependencyManager` validates this.
    c. The prompt for this step is generic, but the `inject: true` feature in `depends_on` automatically prepends the path to `system_design.md`, providing the necessary context to the agent.
    d. The "Engineer" agent reads the task file and the architect's artifact and performs its work.

## 5. Non-Functional Requirements

*   **Testability:** Components must be designed for unit testing. The use of dependency injection is mandated (e.g., the `Engine` receives a `StateManager` and `StepExecutor` instance), allowing for mocks during testing.
*   **Observability:** All significant actions must be logged. Logs must automatically mask any values identified as secrets. The `state.json` file serves as a structured audit log.
*   **Security:**
    *   **Secret Management:** Secrets are passed via environment variables and must never be logged or persisted in the state file.
    *   **Filesystem Safety:** Operations like `--clean-processed` must contain safeguards to prevent execution on paths outside the designated `workspace`.

## 6. Future Considerations (Out of Scope)

*   **Concurrency:** The introduction of `parallel` blocks or event-driven triggers is a potential future extension but is not part of the core design.
*   **Advanced Control Flow:** `while` loops or more complex expression evaluation are not supported initially to maintain simplicity.
*   **Remote Execution:** The current design assumes local command execution. An extension could introduce runners for remote execution (e.g., via SSH or containers).
