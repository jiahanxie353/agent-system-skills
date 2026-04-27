# Python Comments and Docstrings

This document is the detailed comments and docstrings reference for Python code.


## Core Principle

Comments and docstrings should explain intent, contract, behavior, and non-obvious constraints.

They should not narrate obvious Python syntax.

```python
# PASS: GOOD: Explains why this strategy exists.

# Use exponential backoff to avoid overwhelming the API during outages.
delay = min(1000 * (2 ** retry_count), 30_000)


# PASS: GOOD: Explains intentional mutation.

# Deliberately mutate in place here to avoid copying a large list.
items.append(new_item)


# FAIL: BAD: Merely restates the code.

# Increment counter by 1.
count += 1


# FAIL: BAD: Merely restates the code.

# Set name to user's name.
name = user.name
```

If a future reviewer would ask "why did you do this?", write the answer near the code.

If the comment only says what the next line already says, remove it.

## Docstrings

A docstring is a string literal that appears as the first statement in a package,
module, class, function, or method.

That string becomes the object's `__doc__` attribute and can be extracted by
documentation tools.

Use triple double quotes for all docstrings.

```python
def resolve_tool_name(raw_name: str) -> str:
    """Normalize a tool name for registry lookup."""
    ...
```

Use raw triple double quotes when the docstring contains backslashes.

```python
def regex_pattern() -> str:
    r"""Return a pattern matching paths like C:\Users\name."""
    ...
```

## When Docstrings Are Required

Write docstrings for:

- public modules
- public packages
- public classes
- public functions
- public methods
- public properties
- public exceptions
- functions or methods with non-obvious logic
- functions or methods with nontrivial side effects
- functions or methods large enough that their behavior is not obvious
- scripts intended to be run directly from the command line

Docstrings are optional for very small private helpers when the name and type
hints already fully explain the behavior.

```python
# Usually fine without a docstring.

def _is_retryable(error: Exception) -> bool:
    return isinstance(error, TRANSIENT_ERRORS)
```

But if a private helper encodes important policy, add a docstring.

```python
def _is_retryable(error: Exception) -> bool:
    """Return whether retrying this error preserves workflow correctness."""
    ...
```

## When Docstrings Are Not Useful

Do not write docstrings that simply repeat the function or class name.

```python
# FAIL: BAD

def get_user_name(user: User) -> str:
    """Gets the user name."""
    return user.name
```

Prefer no docstring over a useless docstring.

If the function is public, make the docstring meaningful.

```python
# PASS: GOOD

def get_user_name(user: User) -> str:
    """Return the display name used in audit logs."""
    return user.name
```

## One-Line Docstrings

Use one-line docstrings for simple cases.

Rules:

- use triple double quotes
- keep the opening and closing quotes on the same line
- write a phrase ending with a period, question mark, or exclamation point
- describe the function's effect or returned value
- do not restate the function signature
- do not add blank lines before or after the docstring inside the function body

```python
def default_retry_budget() -> int:
    """Return the default number of retry attempts."""
    return 3
```

Do not write signature-style docstrings.

```python
# FAIL: BAD

def resolve_tool_name(raw_name: str) -> str:
    """resolve_tool_name(raw_name) -> str"""
    ...
```

Prefer a semantic description.

```python
# PASS: GOOD

def resolve_tool_name(raw_name: str) -> str:
    """Normalize a tool name for registry lookup."""
    ...
```

PEP 257 prefers imperative-style one-line docstrings such as "Return..." rather
than "Returns...". Google style allows either descriptive or imperative style.
Choose one style and keep it consistent within a file.

```python
# Imperative style.

def load_config(path: Path) -> Config:
    """Load the runtime configuration from disk."""
    ...


# Descriptive style.

def load_config(path: Path) -> Config:
    """Loads the runtime configuration from disk."""
    ...
```

Do not mix styles in the same file without a reason.

## Multi-Line Docstrings

Use multi-line docstrings when behavior, arguments, return values, side effects,
or exceptions require explanation.

A multi-line docstring has:

1. one-line summary
2. blank line
3. detailed description and optional sections

```python
def retry_with_backoff(operation: Callable[[], T], max_retries: int) -> T:
    """Run an operation with exponential backoff.

    Retries only transient failures. Permanent validation errors are raised
    immediately because retrying them cannot succeed.

    Args:
        operation: The operation to run.
        max_retries: The maximum number of retry attempts.

    Returns:
        The value returned by `operation`.

    Raises:
        RetryLimitExceededError: The operation did not succeed before the
            retry budget was exhausted.
    """
```

If a docstring does not fit on one physical line, place the closing quotes on
a line by themselves.

```python
def create_runtime_context(config: Config) -> RuntimeContext:
    """Create the runtime context used by workflow execution.

    The context owns process-wide services such as tracing, tool routing, and
    permission checks.
    """
```

## Summary Line

The first line of a docstring is the summary line.

Rules:

- keep it to one physical line
- keep it concise
- end with a period, question mark, or exclamation point
- make it useful in generated documentation and index views
- do not repeat that the object is a function, class, or method

```python
# PASS: GOOD

class ToolRegistry:
    """A registry of tools available to the agent runtime."""


# FAIL: BAD

class ToolRegistry:
    """Class that implements a tool registry."""
```

## Function and Method Docstrings

A function or method docstring should give enough information to call the
function correctly without reading the implementation.

Document:

- what the function does
- argument semantics when not obvious
- return value semantics when not obvious
- yielded values for generators
- relevant exceptions
- side effects
- mutation of inputs
- restrictions on when the function can be called
- whether keyword arguments are part of the interface, when relevant

Do not document implementation details unless they affect the caller.

```python
def fetch_task_result(
    store: ResultStore,
    task_id: str,
    require_success: bool = False,
) -> TaskResult | None:
    """Fetch a task result from the result store.

    Args:
        store: The result store to query.
        task_id: The task identifier to look up.
        require_success: If True, return only successful task results.

    Returns:
        The matching task result, or None if no matching result exists.

    Raises:
        ResultStoreError: The result store could not be queried.
    """
```

## Args Section

Use an `Args:` section when argument behavior is not obvious from names and type
hints alone.

List each parameter by name.

```python
def run_workflow(
    workflow: Workflow,
    context: RuntimeContext,
    retry_budget: int,
) -> WorkflowResult:
    """Run a workflow to completion.

    Args:
        workflow: The workflow graph to execute.
        context: Runtime services shared across workflow steps.
        retry_budget: The maximum number of recoverable failures to retry.

    Returns:
        The final workflow result.
    """
```

If a parameter has a type annotation, do not repeat the type unless the type
alone is not enough.

```python
# FAIL: BAD: Repeats obvious type information.

def resolve_tool(name: str) -> Tool:
    """Resolve a tool.

    Args:
        name: A string.
    """
    ...


# PASS: GOOD: Explains semantics.

def resolve_tool(name: str) -> Tool:
    """Resolve a tool by its registered name.

    Args:
        name: The public name used in the tool registry.
    """
    ...
```

Document `*args` and `**kwargs` as `*args` and `**kwargs`.

```python
def call_hook(name: str, *args: object, **kwargs: object) -> object:
    """Call a registered runtime hook.

    Args:
        name: The hook name.
        *args: Positional arguments forwarded to the hook.
        **kwargs: Keyword arguments forwarded to the hook.

    Returns:
        The hook result.
    """
```

Use either two-space or four-space hanging indentation in docstring sections,
but be consistent within a file. Prefer four spaces unless the project already
uses two.

## Returns Section

Use a `Returns:` section when the return value needs explanation.

Omit `Returns:` when:

- the function returns `None`
- the one-line summary already fully explains the return value
- the return value is obvious from the function name and type hints

```python
def build_tool_schema(tool: Tool) -> ToolSchema:
    """Build the JSON schema used to validate calls to a tool."""
    ...
```

Use `Returns:` when semantics matter.

```python
def select_next_task(tasks: Sequence[Task]) -> Task | None:
    """Select the next runnable task.

    Returns:
        The highest-priority task whose dependencies are satisfied, or None if
        no task is currently runnable.
    """
```

For tuple returns, describe the tuple as a tuple rather than pretending each
tuple element is a separate return value.

```python
def split_patch(patch: Patch) -> tuple[PatchHeader, PatchBody]:
    """Split a patch into header and body sections.

    Returns:
        A tuple `(header, body)`, where `header` contains metadata and `body`
        contains the search-replace blocks.
    """
```

## Yields Section

Use `Yields:` for generators.

Document the object yielded by each iteration, not the generator object itself.

```python
def iter_ready_tasks(tasks: Sequence[Task]) -> Iterator[Task]:
    """Iterate over tasks whose dependencies are satisfied.

    Yields:
        Each task that is ready to execute.
    """
```

## Raises Section

Use a `Raises:` section for exceptions that are part of the function's interface.

Document exceptions callers may reasonably need to catch or understand.

```python
def apply_patch(source: str, patch: Patch) -> str:
    """Apply a patch to source text.

    Args:
        source: The original source text.
        patch: The patch to apply.

    Returns:
        The patched source text.

    Raises:
        PatchApplicationError: The patch did not match the source text.
    """
```

Do not document exceptions that only occur when the caller violates the documented
API contract, unless that behavior is itself part of the API.

```python
# Usually unnecessary if `patch` is documented as a Patch.

Raises:
    AttributeError: `patch` does not have a blocks attribute.
```

## Side Effects and Mutation

Document side effects that matter to callers.

This is especially important for:

- mutating arguments
- writing files
- updating global or shared state
- network calls
- subprocess execution
- caching
- logging that affects audit trails
- changing permissions or runtime state

```python
def append_trace_event(trace: ExecutionTrace, event: TraceEvent) -> None:
    """Append an event to an execution trace.

    Mutates `trace` in place so callers sharing the trace observe the update.

    Args:
        trace: The trace to update.
        event: The event to append.
    """
```

If a function returns a new object instead of mutating, make that clear when it
could be confused.

```python
def with_trace_event(trace: ExecutionTrace, event: TraceEvent) -> ExecutionTrace:
    """Return a copy of the trace with one additional event."""
```

## Module Docstrings

Every production module should start with a docstring describing its contents,
purpose, or usage.

A module docstring may include:

- high-level purpose
- important exported classes or functions
- usage examples
- important invariants
- command-line usage for scripts

```python
"""Tool registry and lookup utilities for the agent runtime.

This module defines the registry used by the executor to resolve tool names into
callable tool adapters. It also validates tool schemas before registration.
"""
```

For script files, the module docstring should be usable as a usage message.

```python
"""Run the agent workflow benchmark.

Usage:
    python run_benchmark.py --config CONFIG --output OUTPUT

The command loads a workflow benchmark configuration, runs all benchmark cases,
and writes aggregate metrics to the output path.
"""
```

## Test Module Docstrings

Test modules do not need module-level docstrings when the filename and test names
already explain the purpose.

```python
# No module docstring needed for a straightforward test file:
# test_tool_registry.py
```

Add a test module docstring only when it provides useful extra information, such as:

- unusual setup
- external service dependency
- golden file update command
- non-obvious fixture behavior
- environment variables required to run the test

```python
"""Tests for workflow replay using golden traces.

Update the golden traces by running:

    python -m tests.update_golden_traces
"""
```

Avoid docstrings that add no information.

```python
# FAIL: BAD

"""Tests for tool_registry."""
```

## Class Docstrings

A class docstring should describe what an instance of the class represents.

It should not say "Class that...".

```python
# PASS: GOOD

class ToolRegistry:
    """A registry of tools available to the agent runtime."""


# FAIL: BAD

class ToolRegistry:
    """Class that stores tools."""
```

Document public attributes in an `Attributes:` section when the class exposes
public data attributes.

```python
class WorkflowResult:
    """The result of executing a workflow.

    Attributes:
        success: Whether the workflow completed successfully.
        output: The final artifact produced by the workflow, if any.
        failure_reason: The reason execution failed, if available.
    """
```

For dataclasses, class docstrings are useful when fields have semantics that are
not obvious from names and type hints.

```python
@dataclass(frozen=True)
class RetryPolicy:
    """Retry policy for recoverable workflow failures.

    Attributes:
        max_attempts: Maximum number of attempts including the initial attempt.
        backoff_seconds: Delay before the next attempt after a failure.
    """

    max_attempts: int
    backoff_seconds: float
```

## Exception Class Docstrings

Exception class docstrings should describe what the exception represents.

```python
# PASS: GOOD

class MissingArtifactError(Exception):
    """A required workflow artifact is unavailable."""


class InvalidToolCallError(Exception):
    """A tool call does not satisfy its schema."""
```

Avoid phrasing exception docstrings as "Raised when..." unless that wording is
more precise than describing the condition itself.

```python
# FAIL: BAD

class MissingArtifactError(Exception):
    """Raised when an artifact is missing."""
```

## __init__ Docstrings

Public constructors should be documented when initialization parameters are not
obvious.

The class docstring describes what the object represents.

The `__init__` docstring describes initialization arguments and construction-time
behavior.

```python
class TraceStore:
    """Persistent storage for execution traces."""

    def __init__(self, root: Path, flush_on_write: bool = False) -> None:
        """Initialize the trace store.

        Args:
            root: Directory where trace files are stored.
            flush_on_write: If True, flush each event to disk immediately.
        """
```

If a dataclass constructor is obvious from field names and the class docstring
documents field semantics, an explicit `__init__` docstring is not needed.

## Property Docstrings

A property docstring should read like an attribute description.

```python
class ExecutionTrace:
    @property
    def event_count(self) -> int:
        """The number of events in the trace."""
        return len(self.events)
```

Avoid writing property docstrings as if they were methods.

```python
# FAIL: BAD

@property
def event_count(self) -> int:
    """Returns the number of events in the trace."""
    return len(self.events)
```

Avoid properties for expensive computations. Attribute syntax makes callers
expect access to be cheap.

## Overridden Methods

An overridden method does not need a docstring if:

- it is explicitly decorated with `@override`
- the base method has an adequate docstring
- the override does not materially change the contract

```python
from typing_extensions import override

class ChildTool(BaseTool):
    @override
    def run(self, context: RuntimeContext) -> ToolResult:
        ...
```

Add a docstring when the override changes or refines behavior.

```python
class CachingTool(BaseTool):
    @override
    def run(self, context: RuntimeContext) -> ToolResult:
        """Run the tool, returning a cached result when inputs are unchanged."""
        ...
```

Do not write useless override docstrings.

```python
# FAIL: BAD

class ChildTool(BaseTool):
    @override
    def run(self, context: RuntimeContext) -> ToolResult:
        """See base class."""
        ...
```

The `@override` annotation is a clearer signal when supported by the project.

## Block Comments

Use block comments for tricky or non-obvious code.

Place the comment immediately before the code it explains.

```python
# The scheduler stores tasks in reverse topological order so popping from the
# end returns the next dependency-safe task in O(1).
ready_tasks = build_ready_task_stack(workflow)
```

Use block comments to explain:

- non-obvious algorithms
- important invariants
- surprising control flow
- compatibility constraints
- performance tradeoffs
- security or permission decisions
- intentional deviations from the normal style

```python
# Keep this check before tool resolution. Tool resolution may import plugin code,
# and permission denial should not execute plugin-controlled imports.
if not permissions.allow_tool_call(request):
    raise PermissionDeniedError(request.tool_name)
```

## Inline Comments

Use inline comments sparingly.

They are appropriate for short explanations of non-obvious expressions.

Inline comments should be separated from code by at least two spaces and begin
with `# `.

```python
if value & (value - 1) == 0:  # True if value is 0 or a power of 2.
    ...
```

Avoid inline comments that merely restate the expression.

```python
# FAIL: BAD

count += 1  # Increment count.
```

If an inline comment is long, prefer a block comment above the code.

## Comment Style

Write comments as readable prose.

Prefer:

- correct spelling
- proper punctuation
- normal capitalization
- complete sentences for longer comments
- consistent style within a file

```python
# PASS: GOOD

# The verifier runs before synthesis so invalid patches fail quickly.
run_verifier(patch)


# FAIL: BAD

# verifier before synth cuz invalid patch bad
run_verifier(patch)
```

Short inline comments may be less formal, but they should still be clear.

## Comments vs Docstrings

Use docstrings for public contracts.

Use comments for local implementation details.

A docstring explains how to use an object.

A comment explains why the implementation does something locally.

```python
def run_verifier(patch: Patch) -> VerificationResult:
    """Run fast validation checks for a patch.

    Args:
        patch: The patch to validate.

    Returns:
        The validation result.
    """

    # Run schema validation before syntax validation because malformed patch
    # blocks can produce misleading parser diagnostics.
    validate_patch_schema(patch)
    return validate_patch_syntax(patch)
```

Do not put implementation-only details in docstrings unless callers need to know
them.

```python
# FAIL: BAD: This implementation detail does not belong in the public contract.

def run_verifier(patch: Patch) -> VerificationResult:
    """Run fast validation checks using a temporary list and a local cache."""
    ...
```

## Comments for Intentional Rule Violations

When code intentionally violates a general guideline, explain why.

```python
# Deliberately mutate in place to avoid copying a multi-gigabyte trace.
trace.events.extend(new_events)
```

```python
# This import must stay local because the optional plugin is not installed in
# minimal runtime environments.
from plugin_runtime import PluginExecutor
```

Do not use comments to excuse unclear code that could simply be rewritten.

```python
# FAIL: BAD

# This is complicated, but it works.
x = f(g(h(a, b), c), d)
```

Prefer clearer code.

## TODO Comments

Use TODO comments sparingly and make them actionable.

A good TODO includes:

- owner or issue link when the project requires it
- specific missing work
- reason the work is not done now

```python
# TODO(agent-runtime#42): Replace this linear scan with indexed lookup once
# tool registry persistence lands.
for tool in tools:
    ...
```

Avoid vague TODOs.

```python
# FAIL: BAD

# TODO: fix this later
```

## Agentic-System Documentation Guidance

For agentic systems, docstrings should make boundaries and contracts explicit.

Document:

- what artifact a function consumes
- what artifact it produces
- what contract it enforces
- what state it mutates
- what recovery behavior it triggers
- what failures are expected and recoverable
- what failures should stop execution

```python
def execute_tool_call(
    call: ToolCall,
    registry: ToolRegistry,
    context: RuntimeContext,
) -> ToolResult:
    """Execute a validated tool call.

    Args:
        call: The validated tool call to execute.
        registry: Registry used to resolve the tool adapter.
        context: Runtime services available during tool execution.

    Returns:
        The result returned by the tool adapter.

    Raises:
        UnknownToolError: No registered tool matches `call.tool_name`.
        ToolExecutionError: The tool failed after it was invoked.
    """
```

Prefer docstrings that describe the runtime contract over docstrings that
describe the internal control flow.

```python
# PASS: GOOD

def repair_failed_step(step: WorkflowStep, failure: StepFailure) -> RepairPlan:
    """Create a repair plan for a failed workflow step.

    The returned repair plan must preserve the step's declared artifact contract.
    """


# FAIL: BAD

def repair_failed_step(step: WorkflowStep, failure: StepFailure) -> RepairPlan:
    """Check the failure, call the repair model, and then parse the output."""
```

## Good Examples

### Simple Public Function

```python
def normalize_tool_name(name: str) -> str:
    """Normalize a tool name for registry lookup."""
    return name.strip().lower()
```

### Public Function with Contract Details

```python
def validate_artifact(
    artifact: Artifact,
    contract: ArtifactContract,
) -> ValidationResult:
    """Validate an artifact against its declared contract.

    Args:
        artifact: The artifact produced by a workflow step.
        contract: The contract the artifact must satisfy.

    Returns:
        A validation result containing all contract violations. The result is
        successful when no violations are found.
    """
```

### Function with Side Effects

```python
def save_checkpoint(state: WorkflowState, store: CheckpointStore) -> None:
    """Persist the current workflow state.

    Writes a new checkpoint record to `store`. Existing checkpoints are not
    modified.

    Args:
        state: The workflow state to persist.
        store: The checkpoint store to write to.

    Raises:
        CheckpointWriteError: The checkpoint could not be written.
    """
```

### Class with Attributes

```python
@dataclass(frozen=True)
class ToolCall:
    """A request to invoke a registered tool.

    Attributes:
        tool_name: The public name of the tool to invoke.
        arguments: JSON-like arguments passed to the tool.
        call_id: Stable identifier used for tracing and replay.
    """

    tool_name: str
    arguments: Mapping[str, object]
    call_id: str
```

### Module Docstring

```python
"""Workflow scheduling primitives for the agent runtime.

This module defines the scheduler that selects runnable workflow steps from a
dependency graph. It does not execute tools directly; execution is delegated to
the workflow executor.
"""
```

## Bad Examples

### Obvious Comment

```python
# FAIL: BAD

# Loop over tasks.
for task in tasks:
    run_task(task)
```

### Useless Function Docstring

```python
# FAIL: BAD

def run_task(task: Task) -> TaskResult:
    """Runs task."""
    ...
```

### Signature Repetition

```python
# FAIL: BAD

def parse_trace(path: Path) -> ExecutionTrace:
    """parse_trace(path: Path) -> ExecutionTrace"""
    ...
```

### Misplaced Implementation Detail

```python
# FAIL: BAD

def schedule_workflow(workflow: Workflow) -> Schedule:
    """Schedule a workflow using a heap and a visited set."""
    ...
```

A better docstring documents the external behavior.

```python
# PASS: GOOD

def schedule_workflow(workflow: Workflow) -> Schedule:
    """Return an execution schedule that respects step dependencies."""
    ...
```

## Review Checklist

When reviewing comments and docstrings, check:

- Does every public module, class, function, and method have useful documentation?
- Does the summary line fit on one line and end with punctuation?
- Does the docstring explain behavior and contract instead of repeating the name?
- Are `Args:`, `Returns:`, `Yields:`, `Raises:`, and `Attributes:` used when helpful?
- Are type details omitted when type hints already make them obvious?
- Are side effects and mutations documented?
- Are relevant exceptions documented?
- Are comments explaining why rather than restating what?
- Are tricky implementation details documented near the code?
- Are comments written with clear spelling, grammar, and punctuation?
- Are test module docstrings omitted unless they add real information?
- Are override docstrings omitted when `@override` and the base docstring are sufficient?
- Are agent runtime contracts, artifact boundaries, and state mutations explicit where relevant?
