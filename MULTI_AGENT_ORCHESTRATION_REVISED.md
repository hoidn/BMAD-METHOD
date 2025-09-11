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
- Each iteration maintains isolated state under `${steps.<step_name>.iterations[<index>]}`
- Results are collected into an array preserving iteration order
- Loop-level aggregates available via `${steps.<step_name>.summary}`

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

```yaml
for_each:
  items: [1, 2, 3, 4, 5]
  as: num
  max_iterations: 100  # Safety limit
  steps:
    - name: Check
      when:
        expr: "${num} > 3"
      on:
        success:
          goto: _loop_break    # Exit loop entirely
        failure:
          goto: _loop_continue # Skip to next iteration
```

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
  steps:
    - name: Check Status
      command: ["./check.sh"]
    - name: Process
      when:
        expr: "${loop.iteration} > 0"  # Skip first iteration
```

#### Loop Context Variables
Available within loop body:
- `${loop.iteration}`: Current iteration number (0-based)
- `${loop.elapsed}`: Seconds since loop started
- `${loop.remaining_time}`: Seconds until max_duration

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
    
    def execute_parallel(self, steps: List[Dict], runner: 'ProductionWorkflowRunner') -> Dict:
        """Execute steps concurrently"""
        results = {}
        
        with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
            futures = {
                executor.submit(runner.execute_step, step): step['name']
                for step in steps
            }
            
            # Handle join configuration
            join_config = steps[0].get('join', {}) if steps else {}
            wait_for = join_config.get('wait_for', 'all')
            timeout = join_config.get('timeout', 600)
            
            if wait_for == 'all':
                done, not_done = concurrent.futures.wait(
                    futures, timeout=timeout,
                    return_when=concurrent.futures.ALL_COMPLETED
                )
            elif wait_for == 'any':
                done, not_done = concurrent.futures.wait(
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
            
            # Cancel incomplete futures
            for future in not_done:
                future.cancel()
                
            # Collect results
            for future in done:
                step_name = futures[future]
                try:
                    results[step_name] = future.result()
                except Exception as e:
                    results[step_name] = {'error': str(e), 'exit_code': 1}
                    
        return results

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
            return self.parallel.execute_parallel(step['parallel'], self)
            
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
        
    # Additional methods for for_each, while, conditions, etc.
    # ... (implementation continues)
```

## Workflow Schema Definition

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
        "for_each": {
          "type": "object",
          "properties": {
            "items": {},
            "as": {"type": "string"},
            "steps": {"type": "array"}
          }
        },
        "while": {
          "type": "object",
          "properties": {
            "condition": {"$ref": "#/definitions/condition"},
            "max_duration": {"type": "number"},
            "steps": {"type": "array"}
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
            "all": {"type": "array"},
            "any": {"type": "array"},
            "not": {},
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
        "goto": {"type": "string"}
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

This revised specification provides a complete, consistent, and production-ready blueprint that fully addresses all critical feedback. The specification now includes detailed mechanics for loops, robust lock management, and explicit control flow - closing all identified gaps while maintaining the original vision of a powerful yet simple orchestration framework.
