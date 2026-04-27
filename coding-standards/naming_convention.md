# Python Naming Convention

This document is the detailed naming reference for Python code.

The project follows PEP 8 and Google Python style naming conventions.

The main `SKILL.md` should remain concise and simply refer to this file:

```markdown
## Naming Convention

Follow `naming_convention.md` for Python naming rules.
```

## Canonical Naming Forms

| Entity | Public | Internal |
|---|---|---|
| Package | `lowercase` or `lower_with_under` | |
| Module | `lower_with_under.py` | `_lower_with_under.py` |
| Class | `CapWords` | `_CapWords` |
| Exception | `CapWords`, usually with `Error` suffix when appropriate | |
| Function | `lower_with_under()` | `_lower_with_under()` |
| Method | `lower_with_under()` | `_lower_with_under()` |
| Global constant | `CAPS_WITH_UNDER` | `_CAPS_WITH_UNDER` |
| Global variable | `lower_with_under` | `_lower_with_under` |
| Class variable | `lower_with_under` | `_lower_with_under` |
| Instance variable | `lower_with_under` | `_lower_with_under` |
| Function parameter | `lower_with_under` | |
| Local variable | `lower_with_under` | |
| Type variable | `CapWords`, usually short | `_T`, `_P` when private |

## General Rule

Names should be descriptive.

This applies to:

- packages
- modules
- classes
- exceptions
- functions
- methods
- variables
- attributes
- parameters
- files
- tests
- artifacts
- tools
- contracts
- runtime states

Avoid abbreviations unless they are widely understood by the expected reader.

Do not abbreviate by deleting letters inside a word.

```python
# FAIL: BAD

def calc_dep(g):
    ...


# PASS: GOOD

def calculate_dependencies(graph):
    ...
```

Prefer names whose descriptiveness is proportional to their scope.

Short names are acceptable in narrow scopes. Names used across functions, modules, or public APIs must be explicit.

```python
# PASS: GOOD: Narrow scope.

for i, row in enumerate(rows):
    process_row(i, row)


# FAIL: BAD: Too vague for module-level state.

d = load_task_dependency_graph()


# PASS: GOOD: Descriptive module-level name.

task_dependency_graph = load_task_dependency_graph()
```

## File and Module Names

Python files must:

- use the `.py` extension
- use lowercase names
- use underscores when needed for readability
- never use dashes

Good:

```text
tool_registry.py
agent_runtime.py
state_checkpoint.py
execution_trace.py
```

Bad:

```text
tool-registry.py
AgentRuntime.py
stateCheckpoint.py
execution-trace.py
```

Dashes make files harder to import and test as Python modules.

```python
# This works.

import tool_registry


# This does not work as a normal Python import.

import tool-registry
```

Modules should have short, all-lowercase names. Underscores may be used when they improve readability.

Packages should usually have short, all-lowercase names. Underscores are allowed, but package names should stay simple when possible.

Internal implementation modules may use a leading underscore:

```text
_trace_parser.py
_tool_adapter.py
```

## Names to Avoid

Avoid:

- single-character names outside narrow conventional scopes
- ambiguous single-character names such as `l`, `O`, or `I`
- dashes in package or module names
- custom `__double_leading_and_trailing_underscore__` names
- offensive or exclusionary terms
- names that redundantly encode the variable type

### Avoid Ambiguous Single-Character Names

Never use `l`, `O`, or `I` as single-character variable names.

In many fonts, these are hard to distinguish from `1` and `0`.

```python
# FAIL: BAD

l = [1, 2, 3]
O = 0
I = 1
```

### Avoid Redundant Type Suffixes

Do not needlessly include the container type in the name.

```python
# FAIL: BAD: Redundant type suffix.

id_to_name_dict = {"a1": "planner"}
tool_list = [planner_tool, verifier_tool]
name_string = "planner"


# PASS: GOOD: Name the concept, not the container type.

id_to_name = {"a1": "planner"}
tools = [planner_tool, verifier_tool]
name = "planner"
```

A type suffix can be acceptable when it disambiguates two domain concepts, but not when it merely repeats the Python type.

## Allowed Single-Character Names

Single-character names are allowed for:

- short loop counters: `i`, `j`, `k`
- short iterator values in very small scopes
- exception variables: `e`
- file handles: `f`
- private unconstrained type variables: `_T`, `_P`
- names that match established notation in a reference paper or algorithm

Examples:

```python
# PASS: GOOD: Small loop scope.

for i, task in enumerate(tasks):
    schedule_task(i, task)


# PASS: GOOD: Conventional exception name.

try:
    load_config(path)
except FileNotFoundError as e:
    raise ConfigLoadError(path) from e


# PASS: GOOD: Conventional file handle name.

with open(path) as f:
    content = f.read()
```

Do not abuse short names in nested or long-lived scopes.

```python
# FAIL: BAD: Too many vague names across nested scopes.

for i in items:
    for j in i.children:
        for k in j.children:
            process(k)


# PASS: GOOD: Descriptive names make nesting understandable.

for task in tasks:
    for dependency in task.dependencies:
        for subdependency in dependency.dependencies:
            process_dependency(subdependency)
```

## Public, Internal, and Private Names

Python does not have strict private access control. Naming communicates API intent.

### Public Names

Public names have no leading underscore.

Use public names for APIs that external callers are expected to use.

```python
def resolve_tool_name(raw_name: str) -> str:
    ...
```

### Internal Names

Use one leading underscore for internal functions, methods, variables, attributes, classes, and modules.

```python
def _normalize_tool_name(name: str) -> str:
    ...


class _TraceParser:
    ...
```

A single leading underscore means the name is internal to the module or protected within the class.

Unit tests may access protected constants or helpers from the module under test when needed.

### Double Leading Underscores

Avoid double leading underscores unless name mangling is specifically needed to avoid subclass attribute collisions.

```python
# PASS: GOOD: Normal internal attribute.

class AgentRuntime:
    def __init__(self) -> None:
        self._trace_store = TraceStore()


# AVOID unless subclass name collision is a real concern.

class BaseRuntime:
    def __init__(self) -> None:
        self.__internal_state = {}
```

Double leading underscores make debugging, testing, and subclassing harder. Prefer a single underscore unless there is a concrete subclass-collision concern.

### Dunder Names

Do not invent custom `__dunder__` names.

Names with double leading and trailing underscores are reserved by Python.

```python
# FAIL: BAD

def __run_agent_loop__():
    ...
```

Only implement standard Python protocol methods when appropriate.

```python
# PASS: GOOD: Standard Python protocol method.

class ToolSpec:
    def __repr__(self) -> str:
        return f"ToolSpec(name={self.name!r})"
```

## Reserved Keywords

If a name would collide with a Python reserved keyword, append a trailing underscore instead of using an unclear abbreviation or corrupted spelling.

```python
# PASS: GOOD

def create_node(class_: str) -> Node:
    ...


# FAIL: BAD

def create_node(clss: str) -> Node:
    ...
```

Use `cls` only for class objects, especially as the first argument of a class method.

## Classes

Class names use `CapWords`.

```python
class ToolRegistry:
    ...


class AgentRuntime:
    ...


class ExecutionTrace:
    ...
```

Internal classes may use a leading underscore.

```python
class _TraceParser:
    ...
```

Do not name a module after a class using `CapWords.py`.

```text
# FAIL: BAD

ToolRegistry.py


# PASS: GOOD

tool_registry.py
```

Unlike Java, Python modules may contain multiple related classes and top-level functions. There is no need to create one file per class.

Keep related classes and top-level functions together when that improves cohesion.

## Exceptions

Exception classes use `CapWords`.

Use the `Error` suffix when the exception represents an error.

```python
class InvalidToolCallError(Exception):
    ...


class MissingArtifactError(Exception):
    ...


class RetryLimitExceededError(Exception):
    ...
```

Not every exception-like class must end in `Error`, but error exceptions usually should.

## Functions, Methods, and Variables

Function names, method names, parameters, local variables, global variables, and instance variables use lowercase words separated by underscores.

```python
def resolve_tool_name(raw_name: str) -> str:
    normalized_name = raw_name.strip().lower()
    return normalized_name
```

Use `self` for the first argument to instance methods.

Use `cls` for the first argument to class methods.

```python
class ToolSpec:
    def validate(self) -> None:
        ...

    @classmethod
    def from_dict(cls, data: dict[str, object]) -> "ToolSpec":
        ...
```

Mixed case names such as `mixedCase` should only be used when preserving compatibility with an existing API or local legacy convention.

## Constants

Constants are usually defined at module level and written in all capital letters with underscores separating words.

```python
MAX_RETRY_COUNT = 3
DEFAULT_TIMEOUT_SECONDS = 30
TRACE_BUFFER_SIZE = 1024
```

Internal constants may use a leading underscore.

```python
_DEFAULT_BACKOFF_SECONDS = 0.5
```

Only use constants when the value has semantic meaning or is reused. Do not create constant sprawl for values that are obvious and local.

```python
# PASS: GOOD

MAX_RETRY_COUNT = 3

if retry_count > MAX_RETRY_COUNT:
    raise RetryLimitExceededError()


# Usually unnecessary.

ONE = 1
count += ONE
```

## Type Variables

Type variables introduced by typing should normally use `CapWords`, usually short names.

```python
from typing import TypeVar

T = TypeVar("T")
ResultT = TypeVar("ResultT")
```

Use suffixes such as `_co` and `_contra` for covariant and contravariant type variables.

```python
from typing import TypeVar

ValueT_co = TypeVar("ValueT_co", covariant=True)
KeyT_contra = TypeVar("KeyT_contra", contravariant=True)
```

Private unconstrained type variables may use names such as `_T` or `_P`.

## Public Attributes

For simple public data attributes, expose the attribute directly instead of writing unnecessary getter and setter methods.

```python
from dataclasses import dataclass

@dataclass
class ToolResult:
    success: bool
    output: str


result = ToolResult(success=True, output="done")
print(result.output)
```

Avoid Java-style accessors when a public attribute is enough.

```python
# FAIL: BAD: Unnecessary boilerplate.

class ToolResult:
    def __init__(self, output: str) -> None:
        self._output = output

    def get_output(self) -> str:
        return self._output


# PASS: GOOD: Simple data attribute.

@dataclass
class ToolResult:
    output: str
```

Use `@property` when you need to preserve attribute-style access while adding lightweight behavior.

```python
class ExecutionTrace:
    def __init__(self, events: list[TraceEvent]) -> None:
        self.events = events

    @property
    def event_count(self) -> int:
        return len(self.events)
```

Avoid properties for expensive computations, because callers expect attribute access to be cheap.

```python
# FAIL: BAD: Attribute access looks cheap but performs expensive work.

class Report:
    @property
    def full_timing_analysis(self) -> TimingAnalysis:
        return run_expensive_timing_analysis()
```

## Designing for Inheritance

When designing a class, explicitly decide which attributes and methods are:

- public API
- subclass API
- internal implementation details

If in doubt, choose non-public first. It is easier to make an internal attribute public later than to make a public attribute internal later.

Public attributes should have no leading underscore.

Internal implementation attributes should use a single leading underscore.

Double leading underscores may be used only when avoiding accidental subclass name collisions is important.

```python
class BaseTool:
    def run(self) -> ToolResult:
        """Public API."""
        return self._run_impl()

    def _run_impl(self) -> ToolResult:
        """Subclass API / protected implementation hook."""
        raise NotImplementedError
```

Document subclass hooks clearly.

## Tests

Test names should describe behavior.

Recommended pattern:

```text
test_<unit_under_test>_<state_or_condition>_<expected_behavior>
```

Examples:

```python
def test_scheduler_rejects_cyclic_dependencies() -> None:
    ...


def test_tool_registry_returns_registered_tool() -> None:
    ...


def test_repair_loop_stops_after_retry_limit() -> None:
    ...
```

Avoid vague test names:

```python
# FAIL: BAD

def test_scheduler() -> None:
    ...


def test_case_1() -> None:
    ...


def test_works() -> None:
    ...
```

New unit test files should use PEP 8 compliant lowercase names with underscores.

```text
test_tool_registry.py
test_scheduler.py
test_repair_loop.py
```

## Mathematical and Paper-Derived Notation

For mathematically heavy code, short names are allowed when they match established notation from a paper, algorithm, or domain convention.

When using short mathematical names:

- cite the source in a comment or docstring
- keep the short names narrowly scoped
- prefer descriptive PEP 8 names for public APIs
- suppress lint warnings only in the smallest necessary scope

```python
def compute_attention_scores(q: Tensor, k: Tensor) -> Tensor:
    """Compute attention scores.

    Naming follows standard Transformer notation:
    q = query, k = key.
    """
    return q @ k.transpose(-2, -1)
```

For public APIs, prefer descriptive names:

```python
def compute_attention_scores(query: Tensor, key: Tensor) -> Tensor:
    ...
```

If a paper uses established notation, document the convention near the code.

```python
def update_belief(mu: Array, sigma: Array) -> Array:
    """Update Gaussian belief state.

    Uses notation from the referenced algorithm:
    mu = mean, sigma = covariance.
    """
    ...
```

For linter suppressions, keep the disabled scope narrow.

```python
# pylint: disable=invalid-name
def local_matrix_update(A: Matrix, x: Vector) -> Vector:
    # A and x follow standard linear algebra notation.
    return A @ x
```

Prefer a local suppression over disabling naming rules for an entire file.

## Agentic-System Naming

In an agentic system, names should describe the role, artifact, contract, or runtime state precisely.

Good role names:

```python
planner
executor
verifier
critic
repair_loop
scheduler
orchestrator
```

Good artifact and contract names:

```python
tool_schema
tool_adapter
tool_call
tool_result
artifact_contract
patch_contract
execution_trace
state_checkpoint
quality_metric
```

Good runtime state names:

```python
workflow_state
runtime_context
permission_context
retry_budget
failure_reason
pending_tasks
completed_tasks
```

Avoid vague names:

```python
manager
handler
processor
runner
thing
data
info
stuff
logic
helper
util
```

Generic names are acceptable only when the surrounding context makes the meaning precise.

```python
# PASS: GOOD: Context makes the role clear.

class ToolCallHandler:
    def handle(self, call: ToolCall) -> ToolResult:
        ...


# FAIL: BAD: Too vague at top level.

class Handler:
    ...
```

Prefer names that expose the boundary being implemented.

```python
# PASS: GOOD

class ToolRegistry:
    ...


class ArtifactValidator:
    ...


class WorkflowScheduler:
    ...


class CheckpointStore:
    ...


# FAIL: BAD

class Manager:
    ...


class Processor:
    ...


class System:
    ...
```

## Naming for Agent Harness Components

Use precise names for harness layers and responsibilities.

Prefer:

```python
context_builder
tool_router
execution_orchestrator
state_store
memory_store
trace_recorder
failure_classifier
recovery_policy
evaluation_runner
```

Avoid collapsing too many responsibilities into vague names:

```python
agent_manager
main_controller
runtime_handler
system_processor
```

When a component owns several responsibilities, name the specific role of the current class or function, not the entire subsystem.

```python
# FAIL: BAD: Too broad.

class AgentManager:
    ...


# PASS: GOOD: Names identify separate responsibilities.

class ToolCallRouter:
    ...


class WorkflowExecutor:
    ...


class TraceRecorder:
    ...


class RetryPolicy:
    ...
```

## Naming for Tool and Skill Interfaces

Tool and skill names should reveal the action and artifact.

Prefer verb-object names for operations:

```python
parse_timing_report
run_synthesis
validate_patch
apply_patch
check_equivalence
summarize_failure
```

Prefer noun names for data structures:

```python
TimingReport
SynthesisResult
PatchSet
EquivalenceCheckResult
FailureSummary
```

Avoid vague action names:

```python
do_task
handle_data
process_result
run_step
```

A generic name such as `run` is acceptable inside a class where the context is already explicit.

```python
class SynthesisRunner:
    def run(self, design: Design) -> SynthesisResult:
        ...
```

But at module level, prefer a more specific function name.

```python
# PASS: GOOD

def run_synthesis(design: Design) -> SynthesisResult:
    ...


# FAIL: BAD

def run(design: Design) -> SynthesisResult:
    ...
```

## Naming for Error Handling

Exception names should describe the failed condition.

Good:

```python
InvalidToolCallError
MissingArtifactError
PatchApplicationError
EquivalenceCheckError
RetryLimitExceededError
UnsupportedWorkflowError
```

Bad:

```python
ToolError
RuntimeError
BadThingError
Failure
```

Use a broad exception only when the caller truly cannot respond to more specific failure types.

## Naming for Boolean Values

Boolean variables and functions should read naturally as true or false statements.

Prefer:

```python
is_valid
has_failed
can_retry
should_stop
was_repaired
needs_replanning
```

Avoid ambiguous booleans:

```python
valid
failed
retry
stop
flag
```

Examples:

```python
# PASS: GOOD

if should_stop:
    return

if tool_result.has_failed:
    repair(tool_result)


# FAIL: BAD

if flag:
    return
```

## Naming for Collections

Collection names should usually be plural or describe the mapping relationship.

```python
tasks: list[Task]
pending_tasks: list[Task]
tool_names: set[str]
id_to_tool: dict[str, Tool]
module_to_paths: dict[str, list[Path]]
```

Avoid encoding the collection type unless it is necessary to distinguish concepts.

```python
# FAIL: BAD

task_list: list[Task]
tool_name_set: set[str]
id_to_tool_dict: dict[str, Tool]


# PASS: GOOD

tasks: list[Task]
tool_names: set[str]
id_to_tool: dict[str, Tool]
```

## Naming for Units

When a numeric value has units, include the unit in the name.

```python
timeout_seconds = 30
latency_ms = 10.5
memory_bytes = 4096
clock_period_ns = 10.0
```

Avoid unitless names for values where units matter.

```python
# FAIL: BAD

timeout = 30
latency = 10.5
memory = 4096
clock_period = 10.0
```

## Naming Review Checklist

When reviewing names, check:

- Does the name describe the concept rather than the implementation detail?
- Is the name readable outside the original author’s context?
- Is the name proportional to its scope?
- Does the name follow PEP 8 and Google Python conventions?
- Are abbreviations necessary and familiar?
- Are internal names marked with a single leading underscore?
- Are public APIs more descriptive than local variables?
- Are test names behavior-oriented?
- Are mathematical short names justified by a cited convention?
- Does the name avoid redundant type suffixes?
- Does the name include units when units matter?
- Does a boolean name read naturally as true or false?
- Does a tool, skill, or harness component name reveal its role?
