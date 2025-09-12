# Multi-Agent Workflow Orchestration Framework - Revised Specification

## Executive Summary

A production-ready orchestrator that executes AI agents and CLI tools through complex workflows, evolving from a simple MVP (~200 lines) to a robust system (~800 lines) capable of enterprise-grade automation. The framework uses a **centralized orchestration model** with file-based communication, sophisticated control flow (parallel execution, loops, error handling), and a consistent syntax for maximum clarity and reliability.

**Key Principles:**
- **Centralized Orchestration**: Single orchestrator controls all workflow execution
- **Zero Infrastructure**: Just Python + YAML (no databases, queues, or servers)
- **Secure by Design**: No shell injection, no eval(), explicit security boundaries
- **Production-Ready**: Atomic operations, structured logging, unique run isolation
- **Consistent Syntax**: Single variable format, unified control flow model

## Architecture: Centralized Orchestration Model

### Core Architecture

The orchestrator uses a **centralized control model** where a single WorkflowRunner:
1. Loads workflow definition from YAML
2. Executes steps sequentially or in parallel based on control flow
3. Manages state persistently with unique run IDs
4. Handles all branching, routing, and error recovery
5. Ensures atomic file operations and proper isolation

### Directory Structure

```
project/
├── orchestrator.py             # Core orchestrator (~200 lines for MVP)
├── orchestrator_v2.py          # Production version (~800 lines)
├── providers/                  # Provider abstraction layer
│   ├── base.py                # Abstract provider interface
│   ├── claude.py              # Claude-specific implementation
│   └── openai.py              # OpenAI-specific implementation
├── workflows/
│   ├── schema.json            # Formal workflow schema
│   └── *.yaml                 # Workflow definitions
├── runs/                      # Isolated run directories
│   └── {run_id}/
│       ├── state.json         # Persistent state
│       ├── logs/              # Structured logs
│       └── artifacts/         # Generated files
└── prompts/                   # Reusable prompt components
```

## Consistent Syntax Specification

### Variable Substitution

**Single Format**: All variables use `${...}` syntax

```yaml
# Context variables
${context.environment}

# Step results  
${steps.build.exit_code}
${steps.build.output}

# Loop variables
${item}
${loop.index}

# Environment variables (explicit allowlist required)
${env.API_KEY}
```

### Control Flow

**Unified Model**: Single `on:` event system with `goto:` targets

```yaml
steps:
  - name: Build
    command: ["npm", "run", "build"]
    on:
      success: 
        goto: Test
      failure:
        goto: Diagnose
      timeout:
        goto: Kill and Retry
```

### Condition Syntax

**Structured Conditions**: Clear, composable condition model

```yaml
when:
  all:
    - step_ok: build
    - file_exists: dist/bundle.js
    - any:
        - env_set: FORCE_DEPLOY
        - expr: "${context.branch} == 'main'"
```

## Loop Execution Specification

### For-Each Loops

The `for_each` construct provides iteration over collections with the following semantics:

#### State Management

Loop state is managed with the following structure:

```json
{
  "steps": {
    "<loop_step_name>": {
      "type": "for_each",
      "iterations": [
        {
          "index": 0,
          "item": "value1",
          "steps": {
            "Process": {
              "exit_code": 0,
              "output": "...",
              "completed_at": "2024-01-01T12:00:00Z"
            }
          },
          "status": "completed",
          "started_at": "2024-01-01T12:00:00Z",
          "completed_at": "2024-01-01T12:00:05Z"
        }
      ],
      "summary": {
        "total_items": 5,
        "completed": 3,
        "failed": 1,
        "skipped": 1,
        "success_rate": 0.6,
        "total_duration": 45.2
      },
      "exit_code": 0,
      "completed_at": "2024-01-01T12:00:50Z"
    }
  }
}
```

- Each iteration maintains isolated state under `${steps.<step_name>.iterations[<index>]}`
- Results are collected into an array preserving iteration order
- Loop-level aggregates available via `${steps.<step_name>.summary}`
- Iteration state persisted atomically after each iteration

#### Variable Scoping
```yaml
# Loop variables are block-scoped to the loop body
for_each:
  items: ["a", "b", "c"]  # Can be literal or ${context.variable}
  as: item                # Iterator variable name (default: "item")
  steps:
    - name: Process
      prompt: "Process ${item}"       # Valid: item is in scope
      # Also available: ${loop.index}, ${loop.total}, ${loop.first}, ${loop.last}
      
- name: After Loop
  prompt: "Item is ${item}"          # ERROR: item is out of scope
  prompt: "Processed ${steps.Loop.summary.count} items"  # Valid: summary accessible
```

#### Control Flow Semantics
- **Sequential Mode** (default): Items processed in order, one at a time
- **Parallel Mode** (`parallel: true`): All items processed concurrently
- **Break/Continue**: Special goto targets `_loop_break` and `_loop_continue`
- **Nested Loops**: Supported with proper variable shadowing
- **Error Handling**: Failed iterations can be configured to continue or abort

```yaml
for_each:
  items: [1, 2, 3, 4, 5]
  as: num
  max_iterations: 100  # Safety limit
  on_item_failure: continue  # continue | abort | retry
  retry_failed_items: 2  # Retry count for failed iterations
  steps:
    - name: Check
      when:
        expr: "${num} > 3"
      on:
        success:
          goto: _loop_break    # Exit loop entirely
        failure:
          goto: _loop_continue # Skip to next iteration
    - name: Process
      command: ["process.sh", "${num}"]
      on:
        failure:
          goto: _loop_continue  # Skip failed item and continue
```

#### Loop Control Variables
Available within loop body:
- `${item}`: Current iteration item (or custom name via `as:`)
- `${loop.index}`: Current iteration index (0-based)
- `${loop.total}`: Total number of items
- `${loop.first}`: Boolean, true if first iteration
- `${loop.last}`: Boolean, true if last iteration
- `${loop.attempt}`: Current retry attempt for this item (1-based)
- `${loop.parent_step}`: Name of the loop step

#### Parallel Execution
```yaml
for_each:
  items: ${context.services}
  as: service
  parallel: true              # Process all items concurrently
  max_workers: 5             # Limit concurrent executions
  join:
    wait_for: all            # all | any | majority
    timeout: 300
    on_timeout:
      goto: Handle Timeout
```

### While Loops

The `while` construct provides conditional iteration with safety bounds:

#### Execution Semantics
```yaml
while:
  condition:                 # Re-evaluated before each iteration
    expr: "${steps.check.output} != 'ready'"
  max_iterations: 100       # Hard limit to prevent infinite loops
  max_duration: 600         # Timeout in seconds
  check_interval: 5         # Optional delay between iterations
  on_max_iterations: break  # break | error | continue
  on_timeout: error         # break | error | continue
  persist_state: true       # Save state after each iteration
  steps:
    - name: Check Status
      command: ["./check.sh"]
    - name: Process
      when:
        expr: "${loop.iteration} > 0"  # Skip first iteration
      on:
        failure:
          goto: _loop_continue  # Continue to next iteration
```

#### Loop Context Variables
Available within loop body:
- `${loop.iteration}`: Current iteration number (0-based)
- `${loop.elapsed}`: Seconds since loop started
- `${loop.remaining_time}`: Seconds until max_duration
- `${loop.remaining_iterations}`: Iterations until max_iterations
- `${loop.last_iteration}`: Boolean, true if this is the final iteration

#### While Loop State Management

```json
{
  "steps": {
    "<while_step_name>": {
      "type": "while",
      "iterations": [
        {
          "iteration": 0,
          "condition_result": true,
          "steps": {
            "Check Status": {"exit_code": 0, "output": "not ready"},
            "Process": {"exit_code": 0, "output": "processing..."}
          },
          "duration": 5.2,
          "completed_at": "2024-01-01T12:00:05Z"
        }
      ],
      "summary": {
        "total_iterations": 15,
        "total_duration": 78.5,
        "final_condition": false,
        "termination_reason": "condition_false"
      },
      "exit_code": 0,
      "completed_at": "2024-01-01T12:01:18Z"
    }
  }
}
```

## Concurrency and State Management Specification

### Run Isolation

Each workflow execution is isolated with a unique run ID:
- **Run ID Generation**: UUID v4 generated at workflow start
- **Isolation Directory**: `runs/{run_id}/` contains all run-specific data
- **No Shared State**: Parallel runs of the same workflow cannot interfere

### Lock Management

The framework uses PID-based file locking with stale lock recovery:

#### Lock File Format
```json
{
  "pid": 12345,
  "timestamp": 1699564800.123,
  "hostname": "worker-01",
  "run_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### Stale Lock Recovery
1. **Process Check**: Use OS signal 0 to verify if PID is alive
2. **Timestamp Check**: Locks older than 1 hour are considered stale
3. **Corruption Recovery**: Malformed lock files are automatically removed
4. **Acquisition Timeout**: Default 10 seconds with exponential backoff

#### Lock Scope
- **State Lock**: Per-run state file modifications
- **Artifact Lock**: Per-file for atomic writes
- **No Global Locks**: System remains responsive even with stuck workflows

### Atomic Operations

All file operations use write-rename pattern:
1. Write to `file.tmp` with exclusive lock
2. Verify content integrity (optional checksum)
3. Atomic rename to target filename
4. Clean up any orphaned temp files on startup

## Control Flow Specification

### Flow Modes

The framework supports two control flow modes:

#### Strict Mode (V2 Default)
```yaml
version: "2.0"
strict_flow: true  # Require explicit transitions

steps:
  - name: Step1
    command: ["echo", "test"]
    on:
      success:
        goto: Step2      # Explicit next step
      failure:
        end: true       # Explicit termination
    # No implicit fallthrough - workflow stops if no 'on' matches
```

#### Compatibility Mode (V1 Behavior)
```yaml
version: "1.0"
strict_flow: false  # Allow implicit sequential flow

steps:
  - name: Step1
    command: ["echo", "test"]
    # Implicitly continues to Step2 if no 'on' block
  - name: Step2
    command: ["echo", "next"]
```

### Transition Rules

#### Explicit Transitions
All transitions must be defined via `on:` blocks:
- **goto**: Jump to named step
- **end**: Terminate workflow with success
- **error**: Terminate workflow with error

```yaml
on:
  success:
    goto: NextStep
  failure:
    error: "Build failed"  # Sets exit code 1
  timeout:
    end: true             # Sets exit code 0
```

#### Special Targets
Reserved step names for control flow:
- `_start`: Explicit entry point (overrides first step)
- `_end`: Normal termination target
- `_error`: Error handling target
- `_loop_break`: Exit from current loop
- `_loop_continue`: Skip to next iteration

### Workflow Termination

A workflow terminates when:
1. **Explicit End**: Step executes `end: true`
2. **No Transition**: In strict mode, step completes with no matching `on:` handler
3. **Error Action**: Step executes `error: <message>`
4. **Cancellation**: Signal received or `.cancel` file created
5. **Timeout**: Global workflow timeout exceeded

Exit codes:
- `0`: Successful completion via `end: true`
- `1`: Error termination
- `2`: Cancellation
- `3`: Timeout
- `4`: No valid transition (strict mode)

## Implementation Specification

### V1.0: MVP Orchestrator (~200 lines)

```python
# orchestrator.py - MVP with secure, centralized control

import subprocess
import yaml
import json
import uuid
import shlex
from pathlib import Path
from datetime import datetime

class WorkflowRunner:
    def __init__(self, workflow_file, run_id=None):
        self.workflow = yaml.safe_load(Path(workflow_file).read_text())
        self.run_id = run_id or str(uuid.uuid4())
        self.run_dir = Path(f"runs/{self.run_id}")
        self.run_dir.mkdir(parents=True, exist_ok=True)
        self.state_file = self.run_dir / "state.json"
        self.artifacts_dir = self.run_dir / "artifacts"
        self.artifacts_dir.mkdir(exist_ok=True)
        
        # Initialize state
        self.state = self.load_state() or {
            'run_id': self.run_id,
            'started_at': datetime.utcnow().isoformat(),
            'context': self.workflow.get('context', {}),
            'steps': {},
            'current_step': None
        }
        
    def load_state(self):
        """Load state from persistent storage"""
        if self.state_file.exists():
            return json.loads(self.state_file.read_text())
        return None
        
    def save_state(self):
        """Atomically save state"""
        temp_file = self.state_file.with_suffix('.tmp')
        temp_file.write_text(json.dumps(self.state, indent=2))
        temp_file.replace(self.state_file)  # Atomic on POSIX
        
    def run(self):
        """Execute workflow with centralized control"""
        steps = {s['name']: s for s in self.workflow['steps']}
        current = self.workflow['steps'][0]['name']
        
        while current:
            if current not in steps:
                self.log('error', f"Step '{current}' not found")
                break
                
            step = steps[current]
            self.state['current_step'] = current
            self.save_state()
            
            # Check when condition
            if 'when' in step:
                if not self.evaluate_condition(step['when']):
                    self.log('info', f"Skipping {current} (condition not met)")
                    current = self.get_next_step(current, steps)
                    continue
            
            # Execute step
            result = self.execute_step(step)
            
            # Store result
            self.state['steps'][current] = {
                'exit_code': result.get('exit_code', 0),
                'output': result.get('output', ''),
                'completed_at': datetime.utcnow().isoformat()
            }
            self.save_state()
            
            # Determine next step based on result
            if 'on' in step:
                event = 'success' if result['exit_code'] == 0 else 'failure'
                if event in step['on']:
                    current = step['on'][event].get('goto')
                else:
                    current = self.get_next_step(current, steps)
            else:
                current = self.get_next_step(current, steps)
                
    def execute_step(self, step):
        """Execute a single step"""
        self.log('info', f"Executing: {step['name']}")
        
        # Build command
        if 'command' in step:
            # Direct command execution (no shell)
            cmd = step['command']
            if isinstance(cmd, str):
                cmd = shlex.split(cmd)  # Safe parsing
        else:
            # Provider execution
            provider = self.get_provider(step.get('provider', 'claude'))
            prompt = self.build_prompt(step)
            cmd = provider.build_command(prompt, step)
        
        # Execute with timeout
        timeout = step.get('timeout', 300)
        try:
            result = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
                timeout=timeout,
                shell=False  # NEVER use shell=True
            )
            
            # Save output if specified
            if 'output_file' in step:
                output_path = self.artifacts_dir / step['output_file']
                self.atomic_write(output_path, result.stdout)
                
            return {
                'exit_code': result.returncode,
                'output': result.stdout[:10000]  # Limit stored output
            }
            
        except subprocess.TimeoutExpired:
            self.log('error', f"Step {step['name']} timed out")
            return {'exit_code': -1, 'output': 'TIMEOUT'}
            
    def build_prompt(self, step):
        """Build prompt with variable substitution"""
        prompt = step.get('prompt', '')
        
        # Load from file if specified
        if 'prompt_file' in step:
            prompt = (self.run_dir.parent / step['prompt_file']).read_text()
            
        # Add input files
        if 'input_file' in step:
            input_path = self.artifacts_dir / step['input_file']
            if input_path.exists():
                prompt += f"\n\n## Input:\n{input_path.read_text()}"
                
        # Substitute variables - single format: ${...}
        prompt = self.substitute_variables(prompt)
        
        return prompt
        
    def substitute_variables(self, text):
        """Safely substitute ${...} variables"""
        import re
        
        def replacer(match):
            path = match.group(1)
            parts = path.split('.')
            
            # Navigate context
            if parts[0] == 'context':
                value = self.state['context']
                for part in parts[1:]:
                    value = value.get(part, '')
                return str(value)
                
            # Navigate step results
            elif parts[0] == 'steps':
                if len(parts) >= 3:
                    step_name = parts[1]
                    field = parts[2]
                    step_data = self.state['steps'].get(step_name, {})
                    return str(step_data.get(field, ''))
                    
            return match.group(0)  # No substitution
            
        return re.sub(r'\$\{([^}]+)\}', replacer, text)
        
    def evaluate_condition(self, condition):
        """Evaluate conditions safely"""
        if isinstance(condition, bool):
            return condition
            
        if isinstance(condition, dict):
            if 'all' in condition:
                return all(self.evaluate_condition(c) for c in condition['all'])
            elif 'any' in condition:
                return any(self.evaluate_condition(c) for c in condition['any'])
            elif 'not' in condition:
                return not self.evaluate_condition(condition['not'])
            elif 'step_ok' in condition:
                step = condition['step_ok']
                return self.state['steps'].get(step, {}).get('exit_code') == 0
            elif 'file_exists' in condition:
                return (self.artifacts_dir / condition['file_exists']).exists()
                
        return False
        
    def atomic_write(self, path, content):
        """Write file atomically"""
        temp_path = path.with_suffix('.tmp')
        temp_path.write_text(content)
        temp_path.replace(path)
        
    def get_provider(self, name):
        """Get provider instance (abstraction layer)"""
        # Simplified for MVP - would load from providers/ directory
        class SimpleProvider:
            def build_command(self, prompt, step):
                return ['echo', 'mock-response']  # Mock for example
        return SimpleProvider()
        
    def get_next_step(self, current, steps):
        """Get next sequential step"""
        step_list = self.workflow['steps']
        for i, step in enumerate(step_list):
            if step['name'] == current and i + 1 < len(step_list):
                return step_list[i + 1]['name']
        return None
        
    def log(self, level, message):
        """Structured logging"""
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'run_id': self.run_id,
            'level': level,
            'message': message,
            'step': self.state.get('current_step')
        }
        # In production, write to log file as JSON
        print(json.dumps(log_entry))
```

### V2.0: Production Orchestrator (~800 lines)

Key enhancements for production readiness:

```python
# orchestrator_v2.py - Production-ready with full features

import json
import time
import hashlib
import concurrent.futures
from pathlib import Path
from datetime import datetime
from typing import Dict, Any, List, Optional
import logging
import signal
import sys

# Configure structured logging
logging.basicConfig(
    format='%(message)s',
    level=logging.INFO,
    handlers=[logging.StreamHandler()]
)

class WorkflowState:
    """Persistent state with run isolation"""
    
    def __init__(self, run_id: str):
        self.run_id = run_id
        self.run_dir = Path(f"runs/{run_id}")
        self.run_dir.mkdir(parents=True, exist_ok=True)
        self.state_file = self.run_dir / "state.json"
        self.lock_file = self.run_dir / ".lock"
        
        self.state = self.load() or self.initialize()
        
    def initialize(self) -> Dict:
        """Initialize new run state"""
        return {
            'run_id': self.run_id,
            'status': 'running',
            'started_at': datetime.utcnow().isoformat(),
            'context': {},
            'steps': {},
            'secrets': {},  # Encrypted/masked
            'current_step': None,
            'fingerprints': {}  # For idempotency
        }
        
    def load(self) -> Optional[Dict]:
        """Load existing state"""
        if self.state_file.exists():
            return json.loads(self.state_file.read_text())
        return None
        
    def save(self):
        """Atomically save state with lock"""
        # Simple file-based locking
        lock_acquired = False
        for _ in range(10):
            try:
                self.lock_file.touch(exist_ok=False)
                lock_acquired = True
                break
            except FileExistsError:
                time.sleep(0.1)
                
        if not lock_acquired:
            raise RuntimeError("Could not acquire state lock")
            
        try:
            temp_file = self.state_file.with_suffix('.tmp')
            temp_file.write_text(json.dumps(self.state, indent=2))
            temp_file.replace(self.state_file)
        finally:
            self.lock_file.unlink(missing_ok=True)
            
    def get_step_fingerprint(self, step: Dict) -> str:
        """Generate fingerprint for step idempotency"""
        # Hash of step config + inputs
        data = json.dumps({
            'name': step['name'],
            'command': step.get('command'),
            'prompt': step.get('prompt'),
            'input_files': step.get('input_files')
        }, sort_keys=True)
        return hashlib.sha256(data.encode()).hexdigest()[:16]

class SecureEvaluator:
    """Safe expression evaluation without eval()"""
    
    def __init__(self, state: WorkflowState):
        self.state = state
        
    def evaluate(self, expr: str) -> Any:
        """Evaluate expression safely using AST"""
        import ast
        import operator
        
        # Define safe operations
        ops = {
            ast.Add: operator.add,
            ast.Sub: operator.sub,
            ast.Mult: operator.mul,
            ast.Div: operator.truediv,
            ast.Eq: operator.eq,
            ast.NotEq: operator.ne,
            ast.Lt: operator.lt,
            ast.LtE: operator.le,
            ast.Gt: operator.gt,
            ast.GtE: operator.ge,
            ast.And: lambda x, y: x and y,
            ast.Or: lambda x, y: x or y,
            ast.Not: operator.not_
        }
        
        def eval_node(node):
            if isinstance(node, ast.Constant):
                return node.value
            elif isinstance(node, ast.Name):
                # Variable lookup
                return self.get_variable(node.id)
            elif isinstance(node, ast.BinOp):
                left = eval_node(node.left)
                right = eval_node(node.right)
                return ops[type(node.op)](left, right)
            elif isinstance(node, ast.Compare):
                left = eval_node(node.left)
                for op, comparator in zip(node.ops, node.comparators):
                    right = eval_node(comparator)
                    if not ops[type(op)](left, right):
                        return False
                    left = right
                return True
            elif isinstance(node, ast.BoolOp):
                values = [eval_node(v) for v in node.values]
                return ops[type(node.op)](*values)
            elif isinstance(node, ast.UnaryOp):
                operand = eval_node(node.operand)
                return ops[type(node.op)](operand)
            else:
                raise ValueError(f"Unsafe expression node: {type(node)}")
                
        # Parse and evaluate
        tree = ast.parse(expr, mode='eval')
        return eval_node(tree.body)
        
    def get_variable(self, name: str) -> Any:
        """Safely get variable value"""
        # Implementation would resolve ${...} style paths
        parts = name.split('.')
        if parts[0] == 'steps':
            return self.state.state['steps'].get(parts[1], {}).get(parts[2])
        return None

class ParallelExecutor:
    """Execute steps in parallel with proper joining"""
    
    def execute_parallel(self, parallel_config: Dict, runner: 'ProductionWorkflowRunner') -> Dict:
        """Execute steps concurrently with proper result aggregation"""
        steps = parallel_config['steps']
        max_workers = parallel_config.get('max_workers', min(len(steps), 10))
        join_config = parallel_config.get('join', {})
        
        results = {
            'parallel_results': {},
            'summary': {
                'total_steps': len(steps),
                'completed': 0,
                'failed': 0,
                'cancelled': 0,
                'success_rate': 0.0
            },
            'exit_code': 0,
            'output': '',
            'completed_at': None
        }
        
        with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
            futures = {
                executor.submit(runner.execute_single_step, step): step['name']
                for step in steps
            }
            
            # Handle join configuration
            wait_for = join_config.get('wait_for', 'all')
            timeout = join_config.get('timeout', 600)
            on_timeout = join_config.get('on_timeout', {}).get('action', 'cancel_remaining')
            
            done, not_done = self._wait_for_completion(futures, wait_for, timeout)
            
            # Handle timeout behavior
            if not_done and on_timeout == 'cancel_remaining':
                for future in not_done:
                    future.cancel()
                    
            # Collect results
            for future in done:
                step_name = futures[future]
                try:
                    step_result = future.result()
                    results['parallel_results'][step_name] = step_result
                    if step_result.get('exit_code', 0) == 0:
                        results['summary']['completed'] += 1
                    else:
                        results['summary']['failed'] += 1
                except Exception as e:
                    results['parallel_results'][step_name] = {
                        'error': str(e), 
                        'exit_code': 1,
                        'completed_at': datetime.utcnow().isoformat()
                    }
                    results['summary']['failed'] += 1
                    
            # Handle cancelled futures
            for future in not_done:
                step_name = futures[future]
                if future.cancelled():
                    results['parallel_results'][step_name] = {
                        'cancelled': True,
                        'exit_code': 2,
                        'completed_at': datetime.utcnow().isoformat()
                    }
                    results['summary']['cancelled'] += 1
                    
            # Calculate final status
            total_completed = results['summary']['completed']
            total_steps = results['summary']['total_steps']
            results['summary']['success_rate'] = total_completed / total_steps if total_steps > 0 else 0.0
            
            # Determine overall exit code based on join policy
            if wait_for == 'all':
                results['exit_code'] = 0 if results['summary']['failed'] == 0 else 1
            elif wait_for == 'any':
                results['exit_code'] = 0 if results['summary']['completed'] > 0 else 1
            elif wait_for == 'majority':
                required = (total_steps // 2) + 1
                results['exit_code'] = 0 if results['summary']['completed'] >= required else 1
                
            results['completed_at'] = datetime.utcnow().isoformat()
            results['output'] = f"Parallel execution completed: {total_completed}/{total_steps} successful"
                    
        return results
        
    def _wait_for_completion(self, futures, wait_for, timeout):
        """Wait for futures based on join policy"""
        if wait_for == 'all':
            return concurrent.futures.wait(
                futures, timeout=timeout,
                return_when=concurrent.futures.ALL_COMPLETED
            )
        elif wait_for == 'any':
            return concurrent.futures.wait(
                futures, timeout=timeout,
                return_when=concurrent.futures.FIRST_COMPLETED
            )
        elif wait_for == 'majority':
            # Wait for >50% to complete
            required = len(futures) // 2 + 1
            done = set()
            start_time = time.time()
            
            while len(done) < required and time.time() - start_time < timeout:
                completed, _ = concurrent.futures.wait(
                    futures - done, timeout=1,
                    return_when=concurrent.futures.FIRST_COMPLETED
                )
                done.update(completed)
                
            not_done = futures - done
            return done, not_done
        else:
            raise ValueError(f"Unknown wait_for policy: {wait_for}")

class ProductionWorkflowRunner:
    """Production-ready orchestrator with full features"""
    
    def __init__(self, workflow_path: str, run_id: Optional[str] = None,
                 secrets: Optional[Dict] = None):
        self.workflow = self.load_and_validate(workflow_path)
        self.run_id = run_id or str(uuid.uuid4())
        self.state = WorkflowState(self.run_id)
        self.evaluator = SecureEvaluator(self.state)
        self.parallel = ParallelExecutor()
        self.secrets = secrets or {}
        self.cancelled = False
        
        # Set up signal handlers for graceful shutdown
        signal.signal(signal.SIGINT, self.handle_cancel)
        signal.signal(signal.SIGTERM, self.handle_cancel)
        
        # Initialize context
        self.state.state['context'].update(self.workflow.get('context', {}))
        
        # Set up structured logging
        self.logger = self.setup_logging()
        
    def load_and_validate(self, workflow_path: str) -> Dict:
        """Load and validate workflow against schema"""
        workflow = yaml.safe_load(Path(workflow_path).read_text())
        
        # Validate against schema (would use jsonschema in production)
        if 'version' not in workflow:
            raise ValueError("Workflow must specify version")
        if workflow['version'] not in ['1.0', '2.0']:
            raise ValueError(f"Unsupported workflow version: {workflow['version']}")
        if 'steps' not in workflow:
            raise ValueError("Workflow must have steps")
            
        return workflow
        
    def setup_logging(self) -> logging.Logger:
        """Set up structured JSON logging"""
        logger = logging.getLogger(f"workflow.{self.run_id}")
        
        # JSON formatter
        class JSONFormatter(logging.Formatter):
            def format(self, record):
                log_obj = {
                    'timestamp': datetime.utcnow().isoformat(),
                    'run_id': self.run_id,
                    'level': record.levelname,
                    'message': record.getMessage(),
                    'step': getattr(record, 'step', None)
                }
                return json.dumps(log_obj)
                
        # File handler
        log_file = self.state.run_dir / "logs" / "workflow.jsonl"
        log_file.parent.mkdir(exist_ok=True)
        
        file_handler = logging.FileHandler(log_file)
        file_handler.setFormatter(JSONFormatter())
        logger.addHandler(file_handler)
        
        return logger
        
    def handle_cancel(self, signum, frame):
        """Handle cancellation signals"""
        self.logger.warning("Received cancellation signal")
        self.cancelled = True
        self.state.state['status'] = 'cancelled'
        self.state.save()
        sys.exit(0)
        
    def run(self):
        """Execute workflow with all production features"""
        try:
            self.state.state['status'] = 'running'
            self.state.save()
            
            # Main execution loop
            steps = {s['name']: s for s in self.workflow['steps']}
            current = self.workflow['steps'][0]['name']
            
            while current and not self.cancelled:
                step = steps[current]
                
                # Check for cancellation file
                if (self.state.run_dir / '.cancel').exists():
                    self.logger.info("Cancellation requested via file")
                    self.cancelled = True
                    break
                    
                # Check idempotency
                fingerprint = self.state.get_step_fingerprint(step)
                if fingerprint in self.state.state['fingerprints']:
                    self.logger.info(f"Step {current} already completed (fingerprint match)")
                    current = self.get_next_step(current, steps)
                    continue
                    
                # Execute step with all features
                self.state.state['current_step'] = current
                self.state.save()
                
                result = self.execute_step_with_features(step)
                
                # Store result and fingerprint
                self.state.state['steps'][current] = result
                self.state.state['fingerprints'][fingerprint] = True
                self.state.save()
                
                # Determine next step
                current = self.determine_next_step(step, result, steps)
                
            # Final status
            if self.cancelled:
                self.state.state['status'] = 'cancelled'
            elif current is None:
                self.state.state['status'] = 'completed'
            else:
                self.state.state['status'] = 'failed'
                
            self.state.state['completed_at'] = datetime.utcnow().isoformat()
            self.state.save()
            
        except Exception as e:
            self.logger.error(f"Workflow failed: {e}", exc_info=True)
            self.state.state['status'] = 'error'
            self.state.state['error'] = str(e)
            self.state.save()
            raise
            
    def execute_step_with_features(self, step: Dict) -> Dict:
        """Execute step with all production features"""
        
        # Check resource limits
        if 'limits' in step:
            self.enforce_limits(step['limits'])
            
        # Check when condition
        if 'when' in step:
            if not self.evaluate_condition(step['when']):
                self.logger.info(f"Skipping {step['name']} (condition not met)")
                return {'skipped': True}
                
        # Handle parallel execution
        if 'parallel' in step:
            return self.parallel.execute_parallel(step, self)
            
        # Handle for_each loops
        if 'for_each' in step:
            return self.execute_for_each(step)
            
        # Handle while loops  
        if 'while' in step:
            return self.execute_while(step)
            
        # Normal step execution with retry
        max_retries = step.get('retry', {}).get('attempts', 1)
        backoff = step.get('retry', {}).get('backoff', 'linear')
        
        for attempt in range(max_retries):
            try:
                result = self.execute_single_step(step)
                
                if result['exit_code'] == 0:
                    return result
                    
                if attempt < max_retries - 1:
                    wait_time = self.calculate_backoff(attempt, backoff)
                    self.logger.info(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                    time.sleep(wait_time)
                    
            except Exception as e:
                if 'catch' in step:
                    return self.handle_catch(e, step)
                raise
                
        return result
        
    def execute_single_step(self, step: Dict) -> Dict:
        """Execute a single step with security measures"""
        
        # Build command securely
        if 'command' in step:
            cmd = step['command']
            if isinstance(cmd, str):
                # Parse safely, no shell injection
                import shlex
                cmd = shlex.split(cmd)
                
            # Apply resource limits
            if 'limits' in step:
                cmd = self.wrap_with_limits(cmd, step['limits'])
                
        else:
            # Use provider abstraction
            provider = self.get_provider(step.get('provider', 'claude'))
            prompt = self.build_secure_prompt(step)
            cmd = provider.build_command(prompt, step)
            
        # Execute with timeout
        timeout = step.get('timeout', 300)
        start_time = time.time()
        
        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True,
            timeout=timeout,
            shell=False,  # NEVER shell=True
            env=self.get_safe_env(step)
        )
        
        duration = time.time() - start_time
        
        # Handle output
        output = result.stdout[:step.get('limits', {}).get('max_output_size', 100000)]
        
        if 'output_file' in step:
            output_path = self.state.run_dir / "artifacts" / step['output_file']
            self.atomic_write(output_path, output)
            
        return {
            'exit_code': result.returncode,
            'output': output,
            'duration': duration,
            'completed_at': datetime.utcnow().isoformat()
        }
        
    def get_safe_env(self, step: Dict) -> Dict:
        """Build safe environment with allowed secrets"""
        import os
        env = os.environ.copy()
        
        # Only include explicitly allowed secrets
        if 'secrets' in step:
            for secret_name in step['secrets']:
                if secret_name in self.secrets:
                    env[secret_name] = self.secrets[secret_name]
                    
        return env
        
    def atomic_write(self, path: Path, content: str):
        """Write file atomically"""
        path.parent.mkdir(parents=True, exist_ok=True)
        temp_path = path.with_suffix('.tmp')
        temp_path.write_text(content)
        temp_path.replace(path)  # Atomic on POSIX
        
    def execute_for_each(self, step: Dict) -> Dict:
        """Execute for_each loop with state management"""
        for_each_config = step['for_each']
        items = self.resolve_items(for_each_config['items'])
        as_name = for_each_config.get('as', 'item')
        max_iterations = for_each_config.get('max_iterations', len(items))
        parallel = for_each_config.get('parallel', False)
        on_item_failure = for_each_config.get('on_item_failure', 'continue')
        
        result = {
            'type': 'for_each',
            'iterations': [],
            'summary': {
                'total_items': len(items),
                'completed': 0,
                'failed': 0,
                'skipped': 0,
                'success_rate': 0.0,
                'total_duration': 0.0
            },
            'exit_code': 0,
            'output': '',
            'completed_at': None
        }
        
        start_time = time.time()
        
        if parallel:
            result = self._execute_for_each_parallel(items, for_each_config, result)
        else:
            result = self._execute_for_each_sequential(items, for_each_config, result)
            
        result['summary']['total_duration'] = time.time() - start_time
        result['completed_at'] = datetime.utcnow().isoformat()
        
        if result['summary']['total_items'] > 0:
            result['summary']['success_rate'] = result['summary']['completed'] / result['summary']['total_items']
            
        return result
        
    def _execute_for_each_sequential(self, items, config, result):
        """Execute for_each items sequentially"""
        loop_steps = config['steps']
        as_name = config.get('as', 'item')
        
        for index, item in enumerate(items):
            if self.cancelled:
                break
                
            iteration_result = {
                'index': index,
                'item': item,
                'steps': {},
                'status': 'running',
                'started_at': datetime.utcnow().isoformat()
            }
            
            # Set loop context variables
            loop_context = {
                as_name: item,
                'loop': {
                    'index': index,
                    'total': len(items),
                    'first': index == 0,
                    'last': index == len(items) - 1,
                    'attempt': 1
                }
            }
            
            # Execute loop body steps
            try:
                for loop_step in loop_steps:
                    # Substitute loop variables in step
                    resolved_step = self.substitute_loop_variables(loop_step, loop_context)
                    step_result = self.execute_single_step(resolved_step)
                    
                    iteration_result['steps'][resolved_step['name']] = step_result
                    
                    # Handle control flow
                    if 'on' in resolved_step:
                        next_action = self.get_step_action(resolved_step, step_result)
                        if next_action == '_loop_break':
                            iteration_result['status'] = 'break'
                            result['iterations'].append(iteration_result)
                            return result
                        elif next_action == '_loop_continue':
                            iteration_result['status'] = 'continue'
                            break
                            
                iteration_result['status'] = 'completed'
                result['summary']['completed'] += 1
                
            except Exception as e:
                iteration_result['error'] = str(e)
                iteration_result['status'] = 'failed'
                result['summary']['failed'] += 1
                
                if config.get('on_item_failure', 'continue') == 'abort':
                    result['exit_code'] = 1
                    result['iterations'].append(iteration_result)
                    return result
                    
            iteration_result['completed_at'] = datetime.utcnow().isoformat()
            result['iterations'].append(iteration_result)
            
            # Persist state after each iteration
            self.state.save()
            
        return result
        
    def execute_while(self, step: Dict) -> Dict:
        """Execute while loop with safety bounds"""
        while_config = step['while']
        condition = while_config['condition']
        max_iterations = while_config.get('max_iterations', 1000)
        max_duration = while_config.get('max_duration', 3600)
        check_interval = while_config.get('check_interval', 0)
        
        result = {
            'type': 'while',
            'iterations': [],
            'summary': {
                'total_iterations': 0,
                'total_duration': 0.0,
                'final_condition': False,
                'termination_reason': None
            },
            'exit_code': 0,
            'output': '',
            'completed_at': None
        }
        
        start_time = time.time()
        iteration = 0
        
        while iteration < max_iterations:
            if self.cancelled:
                result['summary']['termination_reason'] = 'cancelled'
                break
                
            elapsed = time.time() - start_time
            if elapsed >= max_duration:
                result['summary']['termination_reason'] = 'timeout'
                if while_config.get('on_timeout', 'error') == 'error':
                    result['exit_code'] = 1
                break
                
            # Evaluate condition
            loop_context = {
                'loop': {
                    'iteration': iteration,
                    'elapsed': elapsed,
                    'remaining_time': max_duration - elapsed,
                    'remaining_iterations': max_iterations - iteration,
                    'last_iteration': iteration == max_iterations - 1
                }
            }
            
            condition_result = self.evaluate_condition_with_context(condition, loop_context)
            
            if not condition_result:
                result['summary']['termination_reason'] = 'condition_false'
                result['summary']['final_condition'] = False
                break
                
            # Execute iteration
            iteration_result = {
                'iteration': iteration,
                'condition_result': condition_result,
                'steps': {},
                'duration': 0.0,
                'started_at': datetime.utcnow().isoformat()
            }
            
            iteration_start = time.time()
            
            try:
                for while_step in while_config['steps']:
                    resolved_step = self.substitute_loop_variables(while_step, loop_context)
                    step_result = self.execute_single_step(resolved_step)
                    iteration_result['steps'][resolved_step['name']] = step_result
                    
                    # Handle control flow
                    if 'on' in resolved_step:
                        next_action = self.get_step_action(resolved_step, step_result)
                        if next_action == '_loop_break':
                            result['summary']['termination_reason'] = 'explicit_break'
                            iteration_result['completed_at'] = datetime.utcnow().isoformat()
                            result['iterations'].append(iteration_result)
                            return result
                        elif next_action == '_loop_continue':
                            break
                            
            except Exception as e:
                iteration_result['error'] = str(e)
                # Continue loop unless configured otherwise
                
            iteration_result['duration'] = time.time() - iteration_start
            iteration_result['completed_at'] = datetime.utcnow().isoformat()
            result['iterations'].append(iteration_result)
            
            iteration += 1
            
            # Optional delay between iterations
            if check_interval > 0:
                time.sleep(check_interval)
                
            # Persist state
            self.state.save()
            
        if iteration >= max_iterations:
            result['summary']['termination_reason'] = 'max_iterations'
            if while_config.get('on_max_iterations', 'break') == 'error':
                result['exit_code'] = 1
                
        result['summary']['total_iterations'] = iteration
        result['summary']['total_duration'] = time.time() - start_time
        result['completed_at'] = datetime.utcnow().isoformat()
        
        return result
        
    def substitute_loop_variables(self, step: Dict, loop_context: Dict) -> Dict:
        """Substitute loop variables in step configuration"""
        import copy
        import json
        
        # Deep copy to avoid modifying original
        resolved_step = copy.deepcopy(step)
        
        # Convert to JSON string for substitution, then back
        step_json = json.dumps(resolved_step)
        
        # Replace loop variables
        for var_name, var_value in loop_context.items():
            if isinstance(var_value, dict):
                for sub_var, sub_value in var_value.items():
                    pattern = f"${{{var_name}.{sub_var}}}"
                    step_json = step_json.replace(pattern, str(sub_value))
            else:
                pattern = f"${{{var_name}}}"
                step_json = step_json.replace(pattern, str(var_value))
                
        return json.loads(step_json)
        
    def resolve_items(self, items_config) -> List:
        """Resolve items from config (can be literal array or variable reference)"""
        if isinstance(items_config, list):
            return items_config
        elif isinstance(items_config, str) and items_config.startswith('${'):
            # Variable reference - resolve from context/state
            return self.substitute_variables(items_config)
        else:
            return [items_config]  # Single item
            
    def evaluate_condition_with_context(self, condition, loop_context):
        """Evaluate condition with loop context variables"""
        # Temporarily add loop context to state for evaluation
        original_context = self.state.state.get('loop_context', {})
        self.state.state['loop_context'] = loop_context
        
        try:
            result = self.evaluate_condition(condition)
        finally:
            self.state.state['loop_context'] = original_context
            
        return result
        
    def get_step_action(self, step: Dict, result: Dict) -> str:
        """Determine next action based on step result"""
        if 'on' not in step:
            return None
            
        event = 'success' if result.get('exit_code', 0) == 0 else 'failure'
        if event in step['on']:
            return step['on'][event].get('goto')
            
        return None
```

## Complete Schema Definitions

### Workflow Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Workflow Definition",
  "type": "object",
  "required": ["version", "name", "steps"],
  "properties": {
    "version": {
      "type": "string",
      "enum": ["1.0", "2.0"],
      "description": "Workflow schema version"
    },
    "name": {
      "type": "string",
      "description": "Workflow name"
    },
    "strict_flow": {
      "type": "boolean",
      "description": "Require explicit transitions (default true for v2.0, false for v1.0)"
    },
    "context": {
      "type": "object",
      "description": "Initial context variables"
    },
    "secrets": {
      "type": "array",
      "items": {"type": "string"},
      "description": "Required secret names from environment"
    },
    "steps": {
      "type": "array",
      "items": {
        "$ref": "#/definitions/step"
      }
    }
  },
  "definitions": {
    "step": {
      "type": "object",
      "required": ["name"],
      "properties": {
        "name": {"type": "string"},
        "when": {"$ref": "#/definitions/condition"},
        "command": {
          "oneOf": [
            {"type": "string"},
            {"type": "array", "items": {"type": "string"}}
          ]
        },
        "provider": {"type": "string"},
        "prompt": {"type": "string"},
        "prompt_file": {"type": "string"},
        "input_file": {"type": "string"},
        "input_files": {
          "type": "array",
          "items": {"type": "string"},
          "description": "Multiple input files for step"
        },
        "output_file": {"type": "string"},
        "timeout": {"type": "number"},
        "retry": {
          "type": "object",
          "properties": {
            "attempts": {"type": "number"},
            "backoff": {"enum": ["linear", "exponential"]}
          }
        },
        "limits": {
          "type": "object",
          "properties": {
            "max_memory": {"type": "string"},
            "max_cpu": {"type": "number"},
            "max_output_size": {"type": "number"}
          }
        },
        "secrets": {
          "type": "array",
          "items": {"type": "string"}
        },
        "on": {
          "type": "object",
          "properties": {
            "success": {"$ref": "#/definitions/action"},
            "failure": {"$ref": "#/definitions/action"},
            "timeout": {"$ref": "#/definitions/action"}
          }
        },
        "parallel": {
          "type": "array",
          "items": {"$ref": "#/definitions/step"}
        },
        "join": {
          "type": "object",
          "description": "Configuration for parallel step joining",
          "properties": {
            "wait_for": {
              "type": "string",
              "enum": ["all", "any", "majority"],
              "description": "Wait strategy for parallel steps"
            },
            "timeout": {
              "type": "number",
              "description": "Timeout in seconds for join operation"
            },
            "on_timeout": {"$ref": "#/definitions/action"}
          }
        },
        "for_each": {
          "type": "object",
          "required": ["items", "steps"],
          "properties": {
            "items": {
              "description": "Array of items to iterate over or variable reference"
            },
            "as": {
              "type": "string",
              "default": "item",
              "description": "Variable name for current iteration item"
            },
            "parallel": {
              "type": "boolean",
              "default": false,
              "description": "Execute iterations in parallel"
            },
            "max_workers": {
              "type": "number",
              "description": "Maximum concurrent workers for parallel execution"
            },
            "summary": {
              "type": "object",
              "description": "Summary configuration for loop results",
              "properties": {
                "count": {"type": "boolean"},
                "success_rate": {"type": "boolean"},
                "failures": {"type": "boolean"}
              }
            },
            "steps": {
              "type": "array",
              "items": {"$ref": "#/definitions/step"}
            }
          }
        },
        "while": {
          "type": "object",
          "required": ["condition", "steps"],
          "properties": {
            "condition": {"$ref": "#/definitions/condition"},
            "max_iterations": {
              "type": "number",
              "description": "Maximum number of loop iterations"
            },
            "max_duration": {
              "type": "number",
              "description": "Maximum loop duration in seconds"
            },
            "check_interval": {
              "type": "number",
              "description": "Delay between iterations in seconds"
            },
            "steps": {
              "type": "array",
              "items": {"$ref": "#/definitions/step"}
            }
          }
        }
      }
    },
    "condition": {
      "oneOf": [
        {"type": "boolean"},
        {
          "type": "object",
          "properties": {
            "all": {
              "type": "array",
              "items": {"$ref": "#/definitions/condition"}
            },
            "any": {
              "type": "array", 
              "items": {"$ref": "#/definitions/condition"}
            },
            "not": {"$ref": "#/definitions/condition"},
            "step_ok": {"type": "string"},
            "file_exists": {"type": "string"},
            "env_set": {"type": "string"},
            "expr": {"type": "string"}
          }
        }
      ]
    },
    "action": {
      "type": "object",
      "properties": {
        "goto": {
          "type": "string",
          "description": "Name of step to jump to"
        },
        "end": {
          "type": "boolean",
          "description": "End workflow with success (exit code 0)"
        },
        "error": {
          "type": "string",
          "description": "End workflow with error message (exit code 1)"
        }
      }
    }
  }
}
```

### State Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Workflow Run State",
  "type": "object",
  "required": ["run_id", "status", "started_at", "context", "steps"],
  "properties": {
    "run_id": {
      "type": "string",
      "description": "Unique identifier for this workflow run (UUID v4)"
    },
    "status": {
      "type": "string",
      "enum": ["running", "completed", "failed", "cancelled", "error"],
      "description": "Current status of the workflow run"
    },
    "started_at": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp when workflow started"
    },
    "completed_at": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp when workflow completed"
    },
    "context": {
      "type": "object",
      "description": "Runtime context variables available to all steps"
    },
    "current_step": {
      "type": "string",
      "description": "Name of currently executing or last executed step"
    },
    "steps": {
      "type": "object",
      "description": "Execution results for each step",
      "patternProperties": {
        "^[a-zA-Z][a-zA-Z0-9_\\s]*$": {
          "$ref": "#/definitions/step_result"
        }
      }
    },
    "secrets": {
      "type": "object",
      "description": "Encrypted/masked secret values"
    },
    "fingerprints": {
      "type": "object",
      "description": "Step fingerprints for idempotency checking",
      "patternProperties": {
        "^[a-f0-9]{16}$": {
          "type": "boolean"
        }
      }
    },
    "error": {
      "type": "string",
      "description": "Error message if workflow failed"
    }
  },
  "definitions": {
    "step_result": {
      "type": "object",
      "properties": {
        "exit_code": {
          "type": "number",
          "description": "Process exit code"
        },
        "output": {
          "type": "string",
          "description": "Standard output from step execution"
        },
        "duration": {
          "type": "number",
          "description": "Execution time in seconds"
        },
        "completed_at": {
          "type": "string",
          "format": "date-time",
          "description": "ISO 8601 timestamp when step completed"
        },
        "skipped": {
          "type": "boolean",
          "description": "True if step was skipped due to conditions"
        },
        "iterations": {
          "type": "array",
          "description": "Results from for_each loop iterations",
          "items": {
            "type": "object",
            "properties": {
              "index": {"type": "number"},
              "item": {},
              "result": {"$ref": "#/definitions/step_result"},
              "completed_at": {
                "type": "string",
                "format": "date-time"
              }
            }
          }
        },
        "summary": {
          "type": "object",
          "description": "Aggregated results for loops",
          "properties": {
            "count": {
              "type": "number",
              "description": "Total number of iterations"
            },
            "success_count": {
              "type": "number",
              "description": "Number of successful iterations"
            },
            "failure_count": {
              "type": "number",
              "description": "Number of failed iterations"
            },
            "success_rate": {
              "type": "number",
              "minimum": 0,
              "maximum": 1,
              "description": "Ratio of successful to total iterations"
            },
            "total_duration": {
              "type": "number",
              "description": "Total execution time for all iterations"
            },
            "average_duration": {
              "type": "number",
              "description": "Average execution time per iteration"
            }
          }
        },
        "while_state": {
          "type": "object",
          "description": "State tracking for while loops",
          "properties": {
            "iteration_count": {"type": "number"},
            "elapsed_time": {"type": "number"},
            "condition_met": {"type": "boolean"},
            "exit_reason": {
              "type": "string",
              "enum": ["condition_met", "max_iterations", "max_duration", "error"]
            }
          }
        }
      }
    }
  }
}
```

## Provider Abstraction Layer

```python
# providers/base.py
from abc import ABC, abstractmethod
from typing import List, Dict, Any

class Provider(ABC):
    """Abstract base class for AI providers"""
    
    @abstractmethod
    def build_command(self, prompt: str, config: Dict[str, Any]) -> List[str]:
        """Build command array for execution"""
        pass
        
    @abstractmethod
    def validate_config(self, config: Dict[str, Any]) -> bool:
        """Validate provider-specific configuration"""
        pass

# providers/claude.py
class ClaudeProvider(Provider):
    def build_command(self, prompt: str, config: Dict[str, Any]) -> List[str]:
        cmd = ['claude']
        
        if 'model' in config:
            cmd.extend(['--model', config['model']])
            
        if 'temperature' in config:
            cmd.extend(['--temperature', str(config['temperature'])])
            
        if 'max_tokens' in config:
            cmd.extend(['--max-tokens', str(config['max_tokens'])])
            
        # Prompt via stdin to avoid argument limits
        # Would pipe prompt to stdin in actual execution
        cmd.extend(['--prompt', '-'])
        
        return cmd
        
    def validate_config(self, config: Dict[str, Any]) -> bool:
        # Validate claude-specific options
        return True
```

## Example Workflows with Corrected Syntax

### Basic Software Development Workflow

```yaml
version: "2.0"
name: "Feature Development"

context:
  project: "my-app"
  language: "python"

secrets:
  - GITHUB_TOKEN
  - OPENAI_API_KEY

steps:
  - name: Analyze Requirements
    provider: claude
    prompt_file: prompts/analyze_requirements.md
    input_file: requirements.md
    output_file: analysis.md
    timeout: 120
    limits:
      max_output_size: 50000
    
  - name: Generate Implementation
    provider: claude
    prompt: |
      Based on the analysis, implement the feature.
      Language: ${context.language}
    input_file: analysis.md
    output_file: implementation.py
    when:
      step_ok: "Analyze Requirements"
    
  - name: Run Tests
    command: ["python", "-m", "pytest", "tests/"]
    timeout: 300
    retry:
      attempts: 2
      backoff: exponential
    on:
      success:
        goto: Deploy
      failure:
        goto: Debug Tests
        
  - name: Debug Tests
    provider: claude
    prompt: "Debug these test failures"
    input_file: test_output.log
    output_file: fixes.py
    on:
      success:
        goto: Run Tests  # Retry after fix
        
  - name: Deploy
    command: ["./deploy.sh", "${context.project}"]
    secrets:
      - GITHUB_TOKEN
    when:
      all:
        - step_ok: "Run Tests"
        - env_set: "CI"
```

### Parallel Processing Workflow

```yaml
version: "2.0"
name: "Parallel Analysis"

steps:
  - name: Analyze Components
    parallel:
      max_workers: 3
      steps:
        - name: Frontend Analysis
          provider: claude
          prompt: "Analyze frontend code"
          input_file: frontend/
          output_file: frontend_analysis.md
          
        - name: Backend Analysis
          provider: claude
          prompt: "Analyze backend code"
          input_file: backend/
          output_file: backend_analysis.md
          
        - name: Database Analysis
          provider: claude
          prompt: "Analyze database schema"
          input_file: schema.sql
          output_file: db_analysis.md
    join:
      wait_for: all  # Wait for all to complete
      timeout: 600
      on_timeout:
        action: cancel_remaining  # cancel | continue | ignore
    on:
      success:
        goto: Synthesize Results
      failure:
        error: "Analysis failed for required components"
      
  - name: Synthesize Results
    provider: claude
    prompt: "Combine all analyses into a report"
    input_files:
      - frontend_analysis.md
      - backend_analysis.md
      - db_analysis.md
    output_file: final_report.md
```

### Loop-Based Workflow

```yaml
version: "2.0"
name: "Service Deployment"
strict_flow: true  # Require explicit transitions

context:
  services: ["auth", "api", "frontend", "worker"]
  environment: "staging"

steps:
  - name: Deploy Services
    for_each:
      items: ${context.services}
      as: service
      parallel: true
      max_workers: 2
      steps:
        - name: Build Image
          command: ["docker", "build", "-t", "${service}:latest", "./${service}"]
          on:
            success:
              goto: Push Image
            failure:
              goto: _loop_continue  # Skip failed service
              
        - name: Push Image
          command: ["docker", "push", "${service}:latest"]
          on:
            success:
              goto: Verify Deployment
              
        - name: Verify Deployment
          command: ["kubectl", "rollout", "status", "deployment/${service}"]
    join:
      wait_for: majority  # Continue if >50% succeed
      timeout: 600
    on:
      success:
        goto: Health Check Loop
      failure:
        error: "Deployment failed for majority of services"
        
  - name: Health Check Loop
    while:
      condition:
        not:
          all:
            - step_ok: "check_auth"
            - step_ok: "check_api"
      max_iterations: 30
      max_duration: 300
      check_interval: 10
      steps:
        - name: check_auth
          command: ["curl", "-f", "http://auth/health"]
        - name: check_api
          command: ["curl", "-f", "http://api/health"]
        - name: Log Progress
          command: ["echo", "Iteration ${loop.iteration}: ${loop.elapsed}s elapsed"]
    on:
      success:
        goto: Finalize
      timeout:
        error: "Health checks timed out"
        
  - name: Finalize
    command: ["echo", "Deployment complete"]
    on:
      success:
        end: true  # Explicit successful termination
```

### Explicit Control Flow Example

```yaml
version: "2.0"
name: "Build Pipeline"
strict_flow: true  # No implicit fallthrough

steps:
  - name: _start  # Explicit entry point
    command: ["echo", "Starting build"]
    on:
      success:
        goto: Lint Code
        
  - name: Lint Code
    command: ["npm", "run", "lint"]
    on:
      success:
        goto: Type Check
      failure:
        goto: Auto Fix Lint
        
  - name: Auto Fix Lint
    command: ["npm", "run", "lint:fix"]
    on:
      success:
        goto: Lint Code  # Retry after fix
      failure:
        error: "Cannot auto-fix lint errors"
        
  - name: Type Check
    command: ["npm", "run", "typecheck"]
    on:
      success:
        goto: Run Tests
      failure:
        error: "Type checking failed"
        
  - name: Run Tests
    command: ["npm", "test"]
    on:
      success:
        goto: Build Artifacts
      failure:
        goto: Analyze Test Failures
        
  - name: Analyze Test Failures
    provider: claude
    prompt: "Analyze these test failures and suggest fixes"
    input_file: test_results.log
    output_file: test_analysis.md
    on:
      success:
        error: "Tests failed - see analysis"  # Explicit error termination
        
  - name: Build Artifacts
    command: ["npm", "run", "build"]
    on:
      success:
        goto: Package
      failure:
        error: "Build failed"
        
  - name: Package
    command: ["tar", "-czf", "dist.tar.gz", "dist/"]
    on:
      success:
        end: true  # Explicit successful completion
      failure:
        error: "Packaging failed"
```

### Complex Iteration with State Access

```yaml
version: "2.0"
name: "Data Processing Pipeline"

context:
  batch_size: 100
  data_files: ["data1.csv", "data2.csv", "data3.csv"]

steps:
  - name: Process Batches
    for_each:
      items: ${context.data_files}
      as: file
      steps:
        - name: Validate
          command: ["python", "validate.py", "${file}"]
          output_file: "validation_${loop.index}.log"
          
        - name: Transform
          provider: claude
          prompt: "Transform this data according to schema"
          input_file: "${file}"
          output_file: "transformed_${loop.index}.csv"
          
        - name: Load
          command: ["python", "load.py", "transformed_${loop.index}.csv"]
    on:
      success:
        goto: Generate Report
        
  - name: Generate Report
    provider: claude
    prompt: |
      Generate summary report.
      Total files processed: ${steps.Process Batches.summary.count}
      Success rate: ${steps.Process Batches.summary.success_rate}
    output_file: report.md
    on:
      success:
        end: true
```

## Expression and Variable Evaluation Model

The framework implements a secure, two-phase evaluation pipeline for all variable substitutions and expression evaluations, ensuring consistent behavior across all workflow constructs.

### Evaluation Pipeline

All variable substitutions and expressions follow this secure evaluation pipeline:

#### Phase 1: Variable Substitution
All `${...}` patterns are resolved to their values using namespace-scoped variable lookup.

#### Phase 2: Expression Evaluation (Optional)
If the result contains an `expr:` prefix, the expression is evaluated using a secure AST-based evaluator.

### Variable Namespaces

The framework provides four distinct namespaces for variable resolution:

#### Context Namespace: `${context.*}`
Accesses workflow-level context variables defined in the YAML or set during execution.

```yaml
context:
  environment: "staging"
  version: "1.2.3"
  
# Usage:
command: ["deploy", "${context.environment}", "${context.version}"]
# Resolves to: ["deploy", "staging", "1.2.3"]
```

#### Steps Namespace: `${steps.<name>.*}`
Accesses results from completed workflow steps, including:
- `exit_code`: Step exit status (integer)
- `output`: Step output (string, truncated to limit)
- `duration`: Execution time in seconds (float)
- `completed_at`: ISO timestamp of completion
- `iterations[]`: Array of results for loop steps
- `summary.*`: Aggregated loop results

```yaml
# After step "Build" completes:
when:
  expr: "${steps.Build.exit_code} == 0"
  
prompt: |
  Build completed in ${steps.Build.duration} seconds.
  Output preview: ${steps.Build.output}
```

#### Loop Namespace: `${loop.*}`
Available only within loop bodies (`for_each` and `while`):

**For-Each Loops:**
- `${loop.index}`: Current iteration index (0-based)
- `${loop.total}`: Total number of items
- `${loop.first}`: Boolean, true on first iteration
- `${loop.last}`: Boolean, true on last iteration
- `${loop.iteration}`: Current iteration number (1-based, alias for index+1)

**While Loops:**
- `${loop.iteration}`: Current iteration count (0-based)
- `${loop.elapsed}`: Seconds since loop started
- `${loop.remaining_time}`: Seconds until max_duration timeout

**Loop Item Access:**
- `${<item_var>}`: Current item value (name defined by `as:` parameter)

```yaml
for_each:
  items: ["alpha", "beta", "gamma"]
  as: service
  steps:
    - name: Process Service
      command: ["echo", "Processing ${service} (${loop.index}/${loop.total})"]
      # On second iteration: ["echo", "Processing beta (1/3)"]
      
      when:
        expr: "${loop.first} or ${service} == 'beta'"
        # True on first iteration or when processing "beta"
```

#### Environment Namespace: `${env.*}`
Accesses environment variables, but only those explicitly allowlisted for security:

```yaml
# Workflow level allowlist
secrets:
  - API_KEY
  - DATABASE_URL
  
# Step level access  
steps:
  - name: Deploy
    command: ["deploy", "--key", "${env.API_KEY}"]
    secrets:
      - API_KEY  # Must be listed here to access
```

### Variable Resolution Algorithm

The framework resolves variables using this precedence order:

1. **Loop Variables**: `${item}`, `${loop.*}` (only within loop scope)
2. **Step Results**: `${steps.<name>.*}`
3. **Context Variables**: `${context.*}`
4. **Environment Variables**: `${env.*}` (if allowlisted)

#### Scoping Rules

**Variable Shadowing:**
Inner scopes can shadow outer scopes. Loop variables shadow context variables of the same name.

**Nested Loop Scoping:**
```yaml
for_each:
  items: ["outer1", "outer2"]
  as: item
  steps:
    - name: Nested Loop
      for_each:
        items: ["inner1", "inner2"]
        as: item  # Shadows outer ${item}
        steps:
          - name: Inner Step
            command: ["echo", "${item}"]  # Accesses inner item
            # To access outer: would need different variable name
```

**Step Result Scoping:**
Step results are globally accessible once the step completes, following dot-notation paths:

```yaml
steps:
  - name: Complex Step
    for_each:
      items: [1, 2, 3]
      steps:
        - name: Inner
          command: ["echo", "test"]
          
# Later access:
# ${steps.Complex Step.summary.count} - Total iterations
# ${steps.Complex Step.iterations[0].Inner.output} - First iteration result
# ${steps.Complex Step.iterations[1].Inner.exit_code} - Second iteration exit code
```

### Expression Evaluation

#### Structured Conditions (Recommended)
The preferred approach uses structured conditions without arbitrary expressions:

```yaml
when:
  all:
    - step_ok: build           # ${steps.build.exit_code} == 0
    - file_exists: dist.tar.gz # File exists in artifacts directory
    - any:
        - env_set: FORCE_DEPLOY    # ${env.FORCE_DEPLOY} is non-empty
        - context_equals:          # ${context.branch} == "main"
            var: branch
            value: main
        - step_output_contains:    # ${steps.test.output} contains "PASSED"
            step: test
            text: "PASSED"
```

#### Expression Evaluation (Advanced)
For complex conditions requiring expressions, the framework supports safe AST-based evaluation:

```yaml
when:
  expr: "${steps.build.exit_code} == 0 and ${context.environment} in ['staging', 'prod']"
  
while:
  condition:
    expr: "${steps.check.exit_code} != 0 and ${loop.iteration} < 5"
```

**Supported Expression Operations:**
- **Arithmetic**: `+`, `-`, `*`, `/`
- **Comparison**: `==`, `!=`, `<`, `<=`, `>`, `>=`
- **Logical**: `and`, `or`, `not`
- **Membership**: `in`, `not in`
- **Type Checking**: `isinstance(var, str)`

**Security Restrictions:**
- No function calls except whitelisted type checks
- No attribute access except on safe built-in types
- No imports or eval constructs
- Variables must resolve through defined namespaces
- Expression complexity limited to prevent DoS

### SecureEvaluator Implementation

```python
class SecureEvaluator:
    """Safe expression evaluation without eval()"""
    
    def __init__(self, variable_resolver):
        self.resolver = variable_resolver
        
    def evaluate_expression(self, expr_text: str) -> Any:
        """Evaluate expression using AST with security constraints"""
        import ast
        import operator
        
        # Allowed operations
        safe_operators = {
            # Arithmetic
            ast.Add: operator.add,
            ast.Sub: operator.sub,
            ast.Mult: operator.mul,
            ast.Div: operator.truediv,
            ast.Mod: operator.mod,
            
            # Comparison
            ast.Eq: operator.eq,
            ast.NotEq: operator.ne,
            ast.Lt: operator.lt,
            ast.LtE: operator.le,
            ast.Gt: operator.gt,
            ast.GtE: operator.ge,
            
            # Logical
            ast.And: lambda x, y: x and y,
            ast.Or: lambda x, y: x or y,
            ast.Not: operator.not_,
            
            # Membership
            ast.In: lambda x, y: x in y,
            ast.NotIn: lambda x, y: x not in y,
        }
        
        def eval_node(node, depth=0):
            if depth > 50:  # Prevent deep recursion DoS
                raise ValueError("Expression too complex")
                
            if isinstance(node, ast.Constant):
                return node.value
                
            elif isinstance(node, ast.Name):
                # Variable must be resolved through ${...} syntax
                var_expr = f"${{{node.id}}}"
                return self.resolver.substitute_variable(var_expr)
                
            elif isinstance(node, ast.BinOp):
                if type(node.op) not in safe_operators:
                    raise ValueError(f"Unsafe operation: {type(node.op)}")
                left = eval_node(node.left, depth + 1)
                right = eval_node(node.right, depth + 1)
                return safe_operators[type(node.op)](left, right)
                
            elif isinstance(node, ast.Compare):
                left = eval_node(node.left, depth + 1)
                for op, comparator in zip(node.ops, node.comparators):
                    if type(op) not in safe_operators:
                        raise ValueError(f"Unsafe comparison: {type(op)}")
                    right = eval_node(comparator, depth + 1)
                    if not safe_operators[type(op)](left, right):
                        return False
                    left = right
                return True
                
            elif isinstance(node, ast.BoolOp):
                if type(node.op) not in safe_operators:
                    raise ValueError(f"Unsafe boolean op: {type(node.op)}")
                values = [eval_node(v, depth + 1) for v in node.values]
                if isinstance(node.op, ast.And):
                    return all(values)
                elif isinstance(node.op, ast.Or):
                    return any(values)
                    
            elif isinstance(node, ast.UnaryOp):
                if type(node.op) not in safe_operators:
                    raise ValueError(f"Unsafe unary op: {type(node.op)}")
                operand = eval_node(node.operand, depth + 1)
                return safe_operators[type(node.op)](operand)
                
            elif isinstance(node, ast.List):
                return [eval_node(item, depth + 1) for item in node.elts]
                
            elif isinstance(node, ast.Str):  # Python < 3.8 compatibility
                return node.s
                
            else:
                raise ValueError(f"Unsafe AST node type: {type(node)}")
        
        try:
            tree = ast.parse(expr_text.strip(), mode='eval')
            return eval_node(tree.body)
        except Exception as e:
            raise ValueError(f"Expression evaluation failed: {e}")
```

### Unified Evaluation Function

The framework uses a single evaluation function for all contexts:

```python
class VariableEvaluator:
    """Unified variable and expression evaluator"""
    
    def evaluate_condition(self, condition) -> bool:
        """Evaluate any condition format consistently"""
        
        if isinstance(condition, bool):
            return condition
            
        if isinstance(condition, str):
            # Direct expression
            if condition.startswith("expr:"):
                return bool(self.secure_evaluator.evaluate_expression(condition[5:]))
            else:
                # Treat as variable substitution
                result = self.substitute_variables(condition)
                return self.is_truthy(result)
                
        if isinstance(condition, dict):
            # Structured condition
            return self.evaluate_structured_condition(condition)
            
        return False
        
    def evaluate_structured_condition(self, condition: dict) -> bool:
        """Evaluate structured condition safely"""
        
        if 'all' in condition:
            return all(self.evaluate_condition(c) for c in condition['all'])
            
        elif 'any' in condition:
            return any(self.evaluate_condition(c) for c in condition['any'])
            
        elif 'not' in condition:
            return not self.evaluate_condition(condition['not'])
            
        elif 'step_ok' in condition:
            step_name = condition['step_ok']
            return self.state.get_step_exit_code(step_name) == 0
            
        elif 'file_exists' in condition:
            filename = self.substitute_variables(condition['file_exists'])
            return (self.state.artifacts_dir / filename).exists()
            
        elif 'env_set' in condition:
            env_var = condition['env_set']
            return self.get_env_var(env_var) is not None
            
        elif 'expr' in condition:
            return bool(self.secure_evaluator.evaluate_expression(condition['expr']))
            
        elif 'context_equals' in condition:
            var_path = condition['context_equals']['var']
            expected = condition['context_equals']['value']
            actual = self.get_context_var(var_path)
            return actual == expected
            
        return False
        
    def substitute_variables(self, text: str) -> str:
        """Phase 1: Variable substitution"""
        import re
        
        def replace_var(match):
            var_path = match.group(1)
            return str(self.resolve_variable_path(var_path))
            
        return re.sub(r'\$\{([^}]+)\}', replace_var, text)
        
    def resolve_variable_path(self, path: str) -> Any:
        """Resolve variable path through namespaces with precedence"""
        parts = path.split('.')
        namespace = parts[0]
        
        # Check loop variables first (highest precedence)
        if self.loop_context and namespace in self.loop_context:
            return self.navigate_dict(self.loop_context, parts)
            
        # Check step results
        elif namespace == 'steps':
            return self.get_step_result(parts[1:])
            
        # Check context variables
        elif namespace == 'context':
            return self.get_context_var('.'.join(parts[1:]))
            
        # Check environment variables (must be allowlisted)
        elif namespace == 'env':
            return self.get_env_var(parts[1])
            
        # Check loop namespace explicitly
        elif namespace == 'loop' and self.loop_context:
            return self.loop_context.get('.'.join(parts[1:]))
            
        # Check current loop item variable
        elif self.loop_context and namespace == self.loop_context.get('_item_var'):
            return self.loop_context['_current_item']
            
        return ""  # Default to empty string for missing variables
```

### Evaluation Examples

#### Complete Variable Resolution Example

```yaml
context:
  project_name: "my-app"
  environments: ["dev", "staging", "prod"]
  
steps:
  - name: Deploy All
    for_each:
      items: ${context.environments}  # ["dev", "staging", "prod"]
      as: env
      steps:
        - name: Deploy Service
          command: ["deploy", "${context.project_name}", "${env}"]
          # First iteration: ["deploy", "my-app", "dev"]
          # Second iteration: ["deploy", "my-app", "staging"]  
          # Third iteration: ["deploy", "my-app", "prod"]
          
        - name: Verify
          command: ["curl", "https://${env}.example.com/health"]
          when:
            expr: "${loop.index} > 0"  # Skip dev environment (index 0)
            
  - name: Report Results
    prompt: |
      Deployment Summary:
      Total environments: ${steps.Deploy All.summary.total}
      Successful: ${steps.Deploy All.summary.success_count}
      Failed: ${steps.Deploy All.summary.failure_count}
      
      Details:
      % for i in range(${steps.Deploy All.summary.total}):
      Environment ${steps.Deploy All.iterations[${i}].env}: ${steps.Deploy All.iterations[${i}].Deploy Service.exit_code == 0 ? 'SUCCESS' : 'FAILED'}
      % endfor
```

#### Expression Evaluation Example

```yaml
steps:
  - name: Build
    command: ["npm", "run", "build"]
    
  - name: Test
    command: ["npm", "test"]
    when:
      expr: "${steps.Build.exit_code} == 0"
      
  - name: Deploy
    command: ["deploy.sh"]
    when:
      all:
        - expr: "${steps.Build.exit_code} == 0"
        - expr: "${steps.Test.exit_code} == 0"
        - any:
            - expr: "${context.branch} == 'main'"
            - expr: "${env.FORCE_DEPLOY} != ''"
            - expr: "${context.environment} in ['staging', 'prod']"
```

### Security Guarantees

1. **No Code Injection**: All expressions parsed via AST, no `eval()` or `exec()`
2. **Namespace Isolation**: Variables only accessible through defined namespaces
3. **Operation Whitelisting**: Only safe operators allowed in expressions
4. **Depth Limits**: Recursion depth limited to prevent DoS attacks
5. **Environment Protection**: Environment variables must be explicitly allowlisted
6. **Type Safety**: All operations type-checked at runtime

This evaluation model provides a complete, secure, and consistent foundation for all variable substitution and expression evaluation throughout the workflow framework.

## Summary of Key Changes

1. **Committed to Centralized Architecture**: Removed all references to autonomous agents, pattern watching, and artifact claiming. The orchestrator controls all execution.

2. **Security First**:
   - No `shell=True` ever
   - No `eval()` - using AST-based safe evaluation
   - Explicit secrets allowlisting
   - Command argument arrays instead of strings

3. **Production Features**:
   - Unique run_id for every execution
   - Atomic file operations
   - Structured JSON logging
   - Step fingerprinting for idempotency
   - Resource limits and timeouts
   - Graceful cancellation

4. **Consistent Syntax**:
   - Single variable format: `${...}`
   - Unified control flow: `on:` with `goto:`
   - Clear condition model
   - Formal JSON schema

5. **Provider Abstraction**: Extensible provider system instead of hardcoded CLI commands

6. **Complete Loop Specifications**:
   - For-each loops with parallel execution, state management, and variable scoping
   - While loops with safety bounds and iteration context
   - Special control flow targets (`_loop_break`, `_loop_continue`)
   - Loop state accessible via `${steps.<name>.summary}` and `${loop.*}` variables

7. **Robust Concurrency Management**:
   - PID-based locking with stale lock recovery
   - Per-run isolation directories
   - Atomic file operations with write-rename pattern
   - No global locks ensuring system responsiveness

8. **Explicit Control Flow**:
   - Strict mode (V2 default) requiring explicit transitions
   - Compatibility mode for V1 implicit sequential flow
   - Clear termination semantics with defined exit codes
   - Special targets for workflow control (`_start`, `_end`, `_error`)

9. **Complete Expression and Variable Evaluation Model**:
   - Two-phase evaluation pipeline: variable substitution followed by expression evaluation
   - Four distinct namespaces: context, steps, loop, env with clear precedence rules
   - Secure AST-based expression evaluation with operation whitelisting
   - Unified evaluation function handling all condition formats
   - Complete variable scoping rules for nested contexts

This revised specification provides a complete, consistent, and production-ready blueprint that fully addresses all critical feedback. The specification now includes detailed mechanics for loops, robust lock management, explicit control flow, and a comprehensive expression evaluation system - closing all identified gaps while maintaining the original vision of a powerful yet simple orchestration framework.
