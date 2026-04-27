# Python Patterns

This document is the detailed Python implementation-pattern reference.

## Core Principles

### Readability Counts

Python prioritizes readability. Code should be obvious and easy to understand.

```python
# PASS: GOOD: Clear and readable.

def get_active_users(users: list[User]) -> list[User]:
    return [user for user in users if user.is_active]


# FAIL: BAD: Clever but unclear.

def get_active_users(u):
    return [x for x in u if x.a]
```

Prefer code that another developer, future you, or a coding agent can understand
without reconstructing hidden assumptions.

### Explicit Is Better Than Implicit

Avoid hidden behavior. Make configuration, dependencies, and side effects clear.

```python
# PASS: GOOD: Explicit configuration.

import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
)


# FAIL: BAD: Hidden side effects are unclear.

import some_module

some_module.setup()  # What global state does this mutate?
```

### Simple Is Better Than Clever

Prefer straightforward control flow and simple data structures.

Clever code is expensive to review, test, repair, and modify.

```python
# FAIL: BAD: Compact but hard to debug.

result = {k: f(v) for k, v in data.items() if g(k, v) and h(v)}


# PASS: GOOD: Easy to inspect and extend.

result: dict[str, Output] = {}

for key, value in data.items():
    if not should_include_item(key, value):
        continue
    if not is_supported_value(value):
        continue

    result[key] = transform_value(value)
```

### Prefer Boring Python

Use standard library features and common idioms before introducing custom
frameworks or abstractions.

Good Python code often looks boring:

- `dataclass` for plain data
- `pathlib.Path` for paths
- `with` for resources
- specific exceptions
- explicit imports
- type hints on public boundaries
- small functions with clear responsibilities

## Type Hints

### Annotate Public Boundaries

Use type hints for public functions, methods, class attributes, and important
module-level values.

```python
def process_user(
    user_id: str,
    data: Mapping[str, object],
    active: bool = True,
) -> User | None:
    if not active:
        return None
    return User(user_id=user_id, data=data)
```

Private helpers may also be annotated when doing so improves readability or type
checker coverage.

```python
def _normalize_tool_arguments(arguments: Mapping[str, object]) -> dict[str, object]:
    return dict(arguments)
```

### Prefer Modern Built-In Generic Types

For Python 3.9+, use built-in generic collection types.

```python
# PASS: GOOD: Modern style.

def count_items(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}
```

Avoid older `typing.List` and `typing.Dict` unless the project supports older
Python versions that require them.

```python
# Legacy style for older Python versions.

from typing import Dict, List

def count_items(items: List[str]) -> Dict[str, int]:
    return {item: len(item) for item in items}
```

### Use `T | None` for Optional Values

Prefer modern union syntax when supported.

```python
def first(items: Sequence[T]) -> T | None:
    return items[0] if items else None
```

Avoid returning `None` for failure when the caller needs failure detail. Use a
result object or raise a specific exception instead.

```python
# FAIL: BAD: Failure reason is lost.

def load_config(path: Path) -> Config | None:
    ...


# PASS: GOOD: Failure reason is explicit.

def load_config(path: Path) -> Config:
    ...
```

### Type Aliases

Use type aliases for repeated or complex types.

```python
JSONValue = dict[str, object] | list[object] | str | int | float | bool | None

def parse_json(data: str) -> JSONValue:
    return json.loads(data)
```

Use aliases to clarify domain concepts, not to hide every type.

```python
ToolName = str
CallId = str

def resolve_tool(name: ToolName) -> Tool:
    ...
```

### TypeVar and Generics

Use `TypeVar` for generic functions that preserve type relationships.

```python
from typing import TypeVar

T = TypeVar("T")

def first(items: Sequence[T]) -> T | None:
    return items[0] if items else None
```

### Protocol-Based Duck Typing

Use `Protocol` when code depends on behavior rather than concrete classes.

```python
from typing import Protocol

class Renderable(Protocol):
    def render(self) -> str:
        ...

def render_all(items: Iterable[Renderable]) -> str:
    return "\n".join(item.render() for item in items)
```

Protocols are especially useful for agentic systems where tools, stores, model
clients, or evaluators may have multiple implementations.

```python
class ToolAdapter(Protocol):
    def execute(self, call: ToolCall, context: RuntimeContext) -> ToolResult:
        ...
```

### Avoid Overusing `Any`

`Any` disables useful type checking. Use it only at true dynamic boundaries.

Acceptable places for `Any` include:

- raw JSON-like input before validation
- third-party library boundaries with missing types
- plugin interfaces that intentionally accept arbitrary values

Prefer `object` when the code should not assume anything about a value.

```python
# PASS: GOOD: The function accepts arbitrary values but must inspect them safely.

def validate_argument(value: object) -> ValidatedArgument:
    ...


# USE SPARINGLY: The function may perform dynamic operations.

def call_plugin(name: str, payload: Any) -> Any:
    ...
```

## Data Modeling

### Prefer Simple Data Modeling

Use dictionaries for loose, short-lived data.

Use `dataclass`, `NamedTuple`, `TypedDict`, or Pydantic models when structure
matters.

```python
# FAIL: BAD: Passing unstructured dictionaries through important boundaries.

def run_task(config: dict) -> None:
    timeout = config["timeout"]
    retries = config["retries"]
    ...


# PASS: GOOD: Use explicit structure at module or system boundaries.

@dataclass(frozen=True)
class TaskConfig:
    timeout_seconds: float
    max_retries: int


def run_task(config: TaskConfig) -> None:
    ...
```

### Dataclasses

Use dataclasses for data containers with named fields and predictable behavior.

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class User:
    id: str
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    is_active: bool = True
```

Use `field(default_factory=...)` for mutable defaults or dynamic defaults.

```python
@dataclass
class WorkflowState:
    pending_tasks: list[Task] = field(default_factory=list)
    completed_tasks: list[Task] = field(default_factory=list)
```

Never use mutable values directly as dataclass defaults.

```python
# FAIL: BAD

@dataclass
class WorkflowState:
    pending_tasks: list[Task] = []
```

### Frozen Dataclasses

Prefer `dataclass(frozen=True)` for configuration, identity, and value objects.

```python
@dataclass(frozen=True)
class RetryPolicy:
    max_retries: int
    backoff_seconds: float
```

Frozen dataclasses reduce accidental mutation and make object flow easier to
reason about.

### Dataclass Validation

Use `__post_init__` for lightweight validation.

```python
@dataclass(frozen=True)
class RetryPolicy:
    max_retries: int
    backoff_seconds: float

    def __post_init__(self) -> None:
        if self.max_retries < 0:
            raise ValueError("max_retries must be non-negative")
        if self.backoff_seconds < 0:
            raise ValueError("backoff_seconds must be non-negative")
```

For complex validation, consider a dedicated parser/validator or Pydantic model.

### NamedTuple

Use `NamedTuple` for small immutable records where tuple-like behavior is useful.

```python
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float

    def distance(self, other: "Point") -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5
```

### TypedDict

Use `TypedDict` for dictionary-shaped data where keys are part of the API.

```python
from typing import TypedDict

class ToolCallPayload(TypedDict):
    tool_name: str
    arguments: dict[str, object]
    call_id: str
```

Prefer a dataclass when the value is created and used internally as a domain
object. Prefer `TypedDict` when interoperating with JSON-like dictionaries.

### Pydantic or Runtime Validation Models

Use Pydantic or similar runtime validation models at untrusted boundaries:

- user input
- JSON config files
- tool-call arguments
- plugin payloads
- API responses
- LLM structured outputs

Keep domain logic separate from validation models when the model starts carrying
too much behavior.

## Immutability and Mutation

### Prefer Immutability by Default

Python is not immutable by default, so be explicit.

- Avoid mutating inputs unless the function name or documentation makes that behavior explicit
- Never use mutable default arguments
- Prefer `dataclass(frozen=True)` for configuration and value objects
- Return new values when callers may reuse the original
- Mutate in place only when it is the intended API or needed for performance

```python
# FAIL: BAD: Mutable default argument is shared across calls.

def add_item(item: str, items: list[str] = []) -> list[str]:
    items.append(item)
    return items


# PASS: GOOD: Use None for optional mutable inputs.

def add_item(item: str, items: list[str] | None = None) -> list[str]:
    result = [] if items is None else list(items)
    result.append(item)
    return result
```

### Make Mutation Obvious

Use names that reveal mutation.

```python
# PASS: GOOD

def append_trace_event(trace: ExecutionTrace, event: TraceEvent) -> None:
    trace.events.append(event)


# PASS: GOOD

def with_trace_event(trace: ExecutionTrace, event: TraceEvent) -> ExecutionTrace:
    return replace(trace, events=[*trace.events, event])
```

Avoid hidden mutation in functions that sound like pure computations.

```python
# FAIL: BAD: Name sounds pure but mutates the input.

def compute_trace_summary(trace: ExecutionTrace) -> TraceSummary:
    trace.events.sort()
    ...
```

### Copy at Boundaries When Needed

If a function stores a mutable input for later, copy it unless sharing is part of
the contract.

```python
class ToolRegistry:
    def __init__(self, tools: Iterable[Tool]) -> None:
        self._tools = list(tools)
```

## Error Handling

### Prefer Specific Exceptions

Catch specific exceptions. Avoid bare `except`.

```python
# PASS: GOOD

def load_config(path: Path) -> Config:
    try:
        return Config.from_json(path.read_text())
    except FileNotFoundError as e:
        raise ConfigError(f"Config file not found: {path}") from e
    except json.JSONDecodeError as e:
        raise ConfigError(f"Invalid JSON in config: {path}") from e


# FAIL: BAD

def load_config(path: Path) -> Config | None:
    try:
        return Config.from_json(path.read_text())
    except:
        return None
```

### Use Exception Chaining

Use `raise ... from e` when translating low-level exceptions into domain-specific
exceptions.

```python
def parse_tool_call(data: str) -> ToolCall:
    try:
        raw_call = json.loads(data)
    except json.JSONDecodeError as e:
        raise InvalidToolCallError("Tool call is not valid JSON") from e

    return ToolCall.from_dict(raw_call)
```

Exception chaining preserves the original traceback.

### Custom Exception Hierarchies

Use a custom exception hierarchy when callers need to distinguish project-level
failures.

```python
class AgentRuntimeError(Exception):
    """Base exception for agent runtime errors."""


class ToolExecutionError(AgentRuntimeError):
    """A tool failed during execution."""


class ArtifactValidationError(AgentRuntimeError):
    """An artifact failed contract validation."""
```

Keep the hierarchy small. Do not create custom exceptions that callers cannot
meaningfully handle.

### EAFP vs LBYL

Python often prefers EAFP: "Easier to Ask Forgiveness than Permission."

Use EAFP when:

- the operation is atomic
- the failure mode is clear
- avoiding a race condition matters
- the exception maps cleanly to fallback behavior

```python
# PASS: GOOD: EAFP.

def get_value(mapping: Mapping[str, T], key: str, default: T) -> T:
    try:
        return mapping[key]
    except KeyError:
        return default
```

Use LBYL, "Look Before You Leap," when:

- the check is clearer than exception handling
- failure is expected and frequent
- avoiding exceptions improves readability
- you need to validate multiple conditions and report all errors

```python
# PASS: GOOD: Explicit validation is clearer here.

def validate_retry_policy(policy: RetryPolicy) -> list[str]:
    errors = []
    if policy.max_retries < 0:
        errors.append("max_retries must be non-negative")
    if policy.backoff_seconds < 0:
        errors.append("backoff_seconds must be non-negative")
    return errors
```

Do not treat EAFP as a command to use exceptions everywhere.

### Do Not Swallow Exceptions

Avoid broad exception handlers that hide bugs.

```python
# FAIL: BAD

try:
    run_workflow(workflow)
except Exception:
    pass
```

If you catch a broad exception at a boundary, log it or convert it into a clear
failure artifact.

```python
# PASS: GOOD: Boundary catches and records failure.

try:
    result = run_workflow(workflow)
except Exception as e:
    logger.exception("Workflow execution failed")
    result = WorkflowResult.failed(reason=str(e))
```

## Context Managers

### Use Context Managers for Resources

Use `with` for files, locks, database transactions, temporary directories, and
other resources that must be cleaned up.

```python
# PASS: GOOD

def read_config(path: Path) -> str:
    with path.open() as f:
        return f.read()


# FAIL: BAD

def read_config(path: Path) -> str:
    f = path.open()
    try:
        return f.read()
    finally:
        f.close()
```

Prefer `pathlib.Path` methods for file paths.

```python
def read_config(path: Path) -> str:
    return path.read_text()
```

### Custom Context Managers

Use `contextlib.contextmanager` for simple context managers.

```python
from contextlib import contextmanager
from time import perf_counter
from collections.abc import Iterator

@contextmanager
def timer(name: str) -> Iterator[None]:
    start = perf_counter()
    try:
        yield
    finally:
        elapsed = perf_counter() - start
        print(f"{name} took {elapsed:.4f} seconds")
```

Use class-based context managers when stateful setup/teardown behavior is more
complex.

```python
class DatabaseTransaction:
    def __init__(self, connection: Connection) -> None:
        self.connection = connection

    def __enter__(self) -> "DatabaseTransaction":
        self.connection.begin_transaction()
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_value: BaseException | None,
        traceback: object | None,
    ) -> bool:
        if exc_type is None:
            self.connection.commit()
        else:
            self.connection.rollback()
        return False
```

Return `False` from `__exit__` when exceptions should not be suppressed.

## Comprehensions and Generators

### Use Comprehensions for Simple Transformations

Comprehensions are good for simple mapping, filtering, and collection building.

```python
names = [user.name for user in users if user.is_active]
id_to_user = {user.id: user for user in users}
active_user_ids = {user.id for user in users if user.is_active}
```

### Avoid Overly Clever Comprehensions

Use loops when logic has branches, side effects, or multiple steps.

```python
# FAIL: BAD: Too much logic packed into one expression.

active_admin_names = [
    user.name.strip().lower()
    for user in users
    if user.is_active and user.role == "admin" and user.name is not None
]


# PASS: GOOD: Split when readability improves.

active_admin_names: list[str] = []

for user in users:
    if not user.is_active:
        continue
    if user.role != "admin":
        continue
    if user.name is None:
        continue

    active_admin_names.append(user.name.strip().lower())
```

### Use Generator Expressions for Lazy Evaluation

Use generator expressions when the result is consumed immediately.

```python
total = sum(x * x for x in range(1_000_000))
```

Avoid creating unnecessary intermediate lists.

```python
# FAIL: BAD

total = sum([x * x for x in range(1_000_000)])
```

### Use Generator Functions for Streaming Data

Use generator functions for large or lazy data sources.

```python
from collections.abc import Iterator

def read_lines(path: Path) -> Iterator[str]:
    with path.open() as f:
        for line in f:
            yield line.strip()
```

## Control Flow and Structure

### Use Guard Clauses to Reduce Nesting

Python indentation makes deep nesting expensive to read.

```python
# FAIL: BAD: Excessive nesting.

def update_market(user: User | None, market: Market | None) -> None:
    if user is not None:
        if user.is_admin:
            if market is not None:
                if market.is_active:
                    if user.can_update_market:
                        apply_market_update(market)


# PASS: GOOD: Guard clauses keep the main path flat.

def update_market(user: User | None, market: Market | None) -> None:
    if user is None:
        return
    if not user.is_admin:
        return
    if market is None:
        return
    if not market.is_active:
        return
    if not user.can_update_market:
        return

    apply_market_update(market)
```

If checks represent invalid programmer input or invalid API use, raise exceptions
instead of silently returning.

```python
def update_market(user: User, market: Market) -> None:
    if not user.is_admin:
        raise PermissionError("Only admins can update markets")
    if not market.is_active:
        raise ValueError("Cannot update an inactive market")

    apply_market_update(market)
```

### Keep Functions Small and Cohesive

Split long functions by responsibility.

```python
# FAIL: BAD: Function doing validation, transformation, persistence,
# logging, and notification all in one place.

def process_market_data(raw_rows: list[dict[str, object]]) -> None:
    # 100 lines of mixed responsibilities.
    ...


# PASS: GOOD: Split by responsibility.

def process_market_data(raw_rows: list[dict[str, object]]) -> None:
    validated_rows = validate_market_data(raw_rows)
    transformed_rows = transform_market_data(validated_rows)
    save_market_data(transformed_rows)
```

A long function is not always wrong, but mixed responsibilities usually are.

### Avoid Magic Numbers

Use named constants for values with domain meaning or repeated use.

```python
# FAIL: BAD: Unexplained numbers.

if retry_count > 3:
    raise RetryLimitExceededError()

time.sleep(0.5)


# PASS: GOOD: Named constants for domain meaning.

MAX_RETRIES = 3
DEBOUNCE_DELAY_SECONDS = 0.5

if retry_count > MAX_RETRIES:
    raise RetryLimitExceededError()

time.sleep(DEBOUNCE_DELAY_SECONDS)
```

Avoid constant sprawl for obvious local values.

## Imports

### Prefer Explicit Imports

Avoid star imports.

```python
# FAIL: BAD: Pollutes the namespace and hides dependencies.

from utils import *


# PASS: GOOD: Explicit imports make dependencies clear.

from utils import normalize_name, validate_email
```

### Import Order

Group imports in this order:

1. standard library
2. third-party packages
3. local application imports

```python
import json
import logging
from pathlib import Path

import requests
from pydantic import BaseModel

from agent_runtime.models import ToolCall
from agent_runtime.registry import ToolRegistry
```

Use an import sorter such as `isort` or `ruff` to enforce import order.

### Avoid Import-Time Side Effects

Module imports should not perform expensive work, start services, modify global
state unexpectedly, or execute workflow logic.

```python
# FAIL: BAD: Starts execution at import time.

run_workflow(load_default_workflow())


# PASS: GOOD

def main() -> None:
    run_workflow(load_default_workflow())

if __name__ == "__main__":
    main()
```

### Prefer Local Imports Only for a Reason

Use local imports when:

- avoiding optional dependency failures
- avoiding circular imports
- reducing startup cost for rarely used functionality

Explain surprising local imports with a comment.

```python
def run_plugin(plugin_name: str) -> PluginResult:
    # Import locally because plugin support is an optional dependency.
    from agent_runtime.plugins import PluginExecutor

    return PluginExecutor().run(plugin_name)
```

## Strings and Paths

### Use f-Strings for Formatting

```python
message = f"Tool {tool_name!r} failed with status {status}"
```

Avoid string concatenation for formatted messages.

```python
# FAIL: BAD

message = "Tool " + tool_name + " failed with status " + str(status)
```

### Use `pathlib.Path`

Prefer `pathlib.Path` over manual string path manipulation.

```python
from pathlib import Path

config_path = root / "configs" / "runtime.json"
config_data = config_path.read_text()
```

Avoid hardcoded separators.

```python
# FAIL: BAD

config_path = root + "/configs/runtime.json"
```

### Build Large Strings Efficiently

Use `"".join(...)` or `StringIO` for repeated string building.

```python
result = "".join(str(item) for item in items)
```

For complex incremental writes:

```python
from io import StringIO

buffer = StringIO()
for item in items:
    buffer.write(format_item(item))
result = buffer.getvalue()
```

Avoid repeated concatenation in loops.

```python
# FAIL: BAD: Can become quadratic.

result = ""
for item in items:
    result += str(item)
```

## Decorators

### Preserve Metadata with `functools.wraps`

Use `functools.wraps` when writing function decorators.

```python
import functools
from collections.abc import Callable
from typing import TypeVar

F = TypeVar("F", bound=Callable[..., object])

def log_call(func: F) -> F:
    @functools.wraps(func)
    def wrapper(*args: object, **kwargs: object) -> object:
        logger.info("Calling %s", func.__name__)
        return func(*args, **kwargs)

    return cast(F, wrapper)
```

### Keep Decorators Simple

Decorators should not hide complex control flow or surprising side effects.

Good decorator use cases:

- tracing
- timing
- caching
- registration
- permission checks at clear boundaries

Avoid decorator stacks that obscure execution order.

```python
# FAIL: BAD: Hard to reason about.

@a
@b
@c
@d
def run_step() -> StepResult:
    ...
```

### Parameterized Decorators

Use parameterized decorators when configuration is needed.

```python
def retry(max_attempts: int) -> Callable[[F], F]:
    def decorator(func: F) -> F:
        @functools.wraps(func)
        def wrapper(*args: object, **kwargs: object) -> object:
            ...
        return cast(F, wrapper)

    return decorator
```

If the decorator becomes complicated, prefer an explicit wrapper object or helper
function.

## Concurrency

### Choose the Right Concurrency Model

Use:

- threads for I/O-bound blocking work
- processes for CPU-bound parallel work
- `async`/`await` for structured concurrent I/O
- plain sequential code when concurrency adds no value

### Threading for I/O-Bound Work

```python
import concurrent.futures

def fetch_all_urls(urls: list[str]) -> dict[str, str]:
    results: dict[str, str] = {}

    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        future_to_url = {
            executor.submit(fetch_url, url): url
            for url in urls
        }

        for future in concurrent.futures.as_completed(future_to_url):
            url = future_to_url[future]
            try:
                results[url] = future.result()
            except FetchError as e:
                results[url] = f"Error: {e}"

    return results
```

Catch specific exceptions when possible. If each task may fail independently,
preserve per-task failure information.

### Multiprocessing for CPU-Bound Work

```python
import concurrent.futures

def process_all(datasets: list[list[int]]) -> list[int]:
    with concurrent.futures.ProcessPoolExecutor() as executor:
        return list(executor.map(process_data, datasets))
```

Be mindful that process pools require picklable inputs and outputs.

### Async/Await for Concurrent I/O

Use `async`/`await` when the libraries involved are async-native.

```python
async def fetch_all(urls: list[str]) -> dict[str, str | Exception]:
    tasks = [fetch_async(url) for url in urls]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return dict(zip(urls, results, strict=True))
```

Do not mix blocking I/O into the event loop without offloading it.

```python
# FAIL: BAD: Blocks the event loop.

async def load_file(path: Path) -> str:
    return path.read_text()
```

Use `asyncio.to_thread` for blocking calls inside async code when needed.

```python
async def load_file(path: Path) -> str:
    return await asyncio.to_thread(path.read_text)
```

### Avoid Shared Mutable State

Shared mutable state makes concurrent code hard to reason about.

Prefer:

- immutable inputs
- queues
- message passing
- per-task result objects
- explicit locks when shared mutation is unavoidable

## Package Organization

### Use a `src/` Layout for Packages

A common project layout:

```text
myproject/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── main.py
│       ├── runtime/
│       │   ├── __init__.py
│       │   └── executor.py
│       ├── tools/
│       │   ├── __init__.py
│       │   └── registry.py
│       └── state/
│           ├── __init__.py
│           └── checkpoint.py
├── tests/
│   ├── conftest.py
│   ├── test_executor.py
│   └── test_registry.py
├── pyproject.toml
├── README.md
└── .gitignore
```

### Keep Modules Cohesive

A module should group related concepts.

Avoid dumping unrelated helpers into broad files like:

```text
utils.py
helpers.py
common.py
misc.py
```

Prefer cohesive modules:

```text
path_utils.py
json_schema.py
trace_parser.py
tool_registry.py
checkpoint_store.py
```

### Use `__init__.py` Deliberately

Use `__init__.py` for package exports only when package-level imports improve
the API.

```python
# mypackage/__init__.py

from mypackage.models import ToolCall, ToolResult
from mypackage.registry import ToolRegistry

__all__ = ["ToolCall", "ToolResult", "ToolRegistry"]
```

Avoid large side effects or heavy imports in `__init__.py`.

## Tooling

### Recommended Commands

Project tooling may vary, but common commands are:

```bash
# Format
ruff format .
# or:
black .

# Sort imports and lint
ruff check --fix .

# Type check
mypy .

# Test
pytest

# Test with coverage
pytest --cov=mypackage --cov-report=term-missing
```

Use project-configured tools over global assumptions. If the repository already
uses `ruff`, do not introduce `black`, `isort`, or `pylint` unless there is a
reason.

### pyproject.toml

Prefer centralizing Python tool configuration in `pyproject.toml`.

Example:

```toml
[project]
name = "mypackage"
version = "1.0.0"
requires-python = ">=3.11"
dependencies = []

[project.optional-dependencies]
dev = [
    "pytest",
    "ruff",
    "mypy",
]

[tool.ruff]
line-length = 88

[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
```

## Performance and Memory

### Prefer Correctness and Clarity First

Do not optimize prematurely.

Optimize when:

- profiling shows a real bottleneck
- the code path is performance critical
- the optimization does not obscure correctness
- the tradeoff is documented

### Use Generators for Large Data

```python
def read_lines(path: Path) -> Iterator[str]:
    with path.open() as f:
        for line in f:
            yield line.rstrip("\n")
```

### Use `__slots__` Selectively

`__slots__` can reduce memory for many small objects, but it also changes class
behavior and flexibility.

Use it only when object count and memory pressure justify it.

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y
```

For most data objects, start with a dataclass.

## Common Anti-Patterns

### Mutable Default Arguments

```python
# FAIL: BAD

def append_to(item: str, items: list[str] = []) -> list[str]:
    items.append(item)
    return items


# PASS: GOOD

def append_to(item: str, items: list[str] | None = None) -> list[str]:
    result = [] if items is None else list(items)
    result.append(item)
    return result
```

### Bare Except

```python
# FAIL: BAD

try:
    risky_operation()
except:
    pass


# PASS: GOOD

try:
    risky_operation()
except SpecificError as e:
    logger.exception("Operation failed")
    raise OperationError("Operation failed") from e
```

### Comparing to None with `==`

```python
# FAIL: BAD

if value == None:
    ...


# PASS: GOOD

if value is None:
    ...
```

### Type Checks with `type(...) == ...`

```python
# FAIL: BAD

if type(obj) == list:
    ...


# PASS: GOOD

if isinstance(obj, list):
    ...
```

Prefer protocols or duck typing when behavior matters more than concrete type.

### Star Imports

```python
# FAIL: BAD

from os.path import *


# PASS: GOOD

from os.path import exists, join
```

### Hidden Global State

```python
# FAIL: BAD

CURRENT_CONTEXT = RuntimeContext()

def run_step(step: Step) -> StepResult:
    return step.run(CURRENT_CONTEXT)
```

Prefer explicit dependencies.

```python
# PASS: GOOD

def run_step(step: Step, context: RuntimeContext) -> StepResult:
    return step.run(context)
```

### Boolean Flags That Change Function Meaning

Avoid functions whose behavior radically changes based on boolean flags.

```python
# FAIL: BAD

def run_workflow(workflow: Workflow, dry_run: bool = False) -> WorkflowResult:
    ...


# PASS: GOOD

def run_workflow(workflow: Workflow) -> WorkflowResult:
    ...


def preview_workflow(workflow: Workflow) -> WorkflowPreview:
    ...
```

Boolean flags are acceptable when they refine behavior rather than create a
separate mode.

### Overly Broad Utility Modules

Avoid putting unrelated functions in `utils.py`.

```text
# FAIL: BAD

utils.py
```

Prefer modules named after cohesive responsibilities.

```text
path_utils.py
json_schema.py
retry_policy.py
trace_parser.py
```

## Agentic-System Python Patterns

### Make Runtime Boundaries Explicit

Agentic systems should expose clear boundaries between:

- planning
- execution
- tool routing
- validation
- repair
- state persistence
- evaluation

Prefer explicit data models at these boundaries.

```python
@dataclass(frozen=True)
class ToolCall:
    tool_name: str
    arguments: Mapping[str, object]
    call_id: str


@dataclass(frozen=True)
class ToolResult:
    call_id: str
    success: bool
    output: str | None
    error: str | None
```

### Separate Validation from Execution

Do not execute unvalidated tool calls.

```python
def execute_tool_call(
    raw_call: Mapping[str, object],
    registry: ToolRegistry,
    context: RuntimeContext,
) -> ToolResult:
    call = validate_tool_call(raw_call)
    tool = registry.resolve(call.tool_name)
    return tool.execute(call, context)
```

### Prefer Result Objects at Runtime Boundaries

For internal pure functions, exceptions are fine.

At workflow boundaries, result objects are often better because the runtime needs
to record, inspect, retry, or repair failures.

```python
@dataclass(frozen=True)
class StepResult:
    step_id: str
    status: StepStatus
    artifact: Artifact | None = None
    failure: StepFailure | None = None
```

### Preserve Traces

Runtime code should produce inspectable traces rather than hiding state in logs.

```python
def record_tool_result(trace: ExecutionTrace, result: ToolResult) -> ExecutionTrace:
    return trace.with_event(ToolResultEvent.from_result(result))
```

### Keep Repair Logic Separate

Do not bury repair behavior inside execution code.

```python
# PASS: GOOD

result = executor.execute(step)

if result.failed:
    repair_plan = repair_planner.plan(step, result.failure)
    return executor.execute_repair(repair_plan)
```

Avoid mixing execution, diagnosis, and repair in one large function.

## Quick Reference

| Pattern | Preferred Practice |
|---|---|
| Type hints | Annotate public boundaries |
| Data containers | Use `dataclass`, `NamedTuple`, `TypedDict`, or Pydantic |
| Immutability | Prefer frozen dataclasses for value/config objects |
| Mutable defaults | Use `None` plus a new object |
| Errors | Catch specific exceptions and use exception chaining |
| Resources | Use context managers |
| Imports | Use explicit imports and standard grouping |
| Comprehensions | Use for simple transformations only |
| Generators | Use for large or lazy data |
| Paths | Use `pathlib.Path` |
| Strings | Use f-strings and `join` |
| Package layout | Prefer cohesive modules and `src/` layout |
| Agent runtime | Make contracts, state, and failure artifacts explicit |

## Review Checklist

When reviewing Python patterns, check:

- Are public boundaries type annotated?
- Are data structures explicit at important boundaries?
- Are mutable defaults avoided?
- Are inputs mutated only when the API makes mutation explicit?
- Are exceptions specific and chained when translated?
- Are context managers used for resources?
- Are comprehensions simple enough to read?
- Are long functions split by responsibility?
- Is deep nesting replaced with guard clauses where appropriate?
- Are magic numbers named when they carry domain meaning?
- Are imports explicit and ordered?
- Are paths handled with `pathlib.Path`?
- Are expensive import-time side effects avoided?
- Are result objects used where workflow/runtime inspection needs them?
- Are agent runtime boundaries visible in the code structure?
