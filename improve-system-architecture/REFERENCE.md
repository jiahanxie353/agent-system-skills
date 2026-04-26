# Reference

Lookup material for the `improve-agent-system-architecture` skill. Use after the skill has identified candidate friction. Keep proposals grounded in code evidence, not generic agent-system advice.

The recurring frame: candidates should expose **split ownership** — one harness responsibility split across code, prompts, state, tests, or adapters without a clear contract surface or single owner, missing ownership is the smell.

## 1. Boundary Categories

Classify each refactor candidate by the hardest boundary it must cross. The category drives the testing strategy.

### 1.1 In-process agent logic

Pure local computation or orchestration with no external side effects.

Examples: prompt construction, schema normalization, skill selection, planning heuristics, artifact transformation, context packing, local state-machine transitions, error classification, acceptance-check composition.

Recommendation: merge scattered helpers into one deep module with a small public interface. Test directly through the boundary.

Good tests: input artifact → output artifact; state before → state after; invalid input → structured validation error; context budget → deterministic context package; trace events → attributed failure category.

### 1.2 Local-substitutable runtime boundary

The module depends on a local resource or runtime that can be replaced in tests.

Examples: filesystem, shell command runner, local database, local vector store, local model stub, in-memory queue, temporary workspace, deterministic clock, local sandbox, local artifact store.

Recommendation: deepen the module when a local stand-in exists. Production uses the real local resource; tests use the stand-in.

Good tests: execute against temporary files; against an in-memory store; with a fake shell or tool runner; with a deterministic model stub; replay a saved trajectory; inject timeouts or malformed local responses.

### 1.3 LLM reasoning boundary

The system depends on nondeterministic model behavior.

Examples: planner LLM, router LLM, critic/verifier LLM, repair LLM, summarizer LLM, tool-selection LLM, code-generation LLM.

Recommendation: separate the model call from the runtime contract. The deep module owns the prompt contract, input construction, output schema, parser, validator, repair behavior, fallback behavior, trace logging, and deterministic test fixtures. Free-form LLM output should not leak deep into the system.

Good tests: golden response parsing; invalid response repair; missing field handling; schema evolution; deterministic fake-model behavior; regression tests from real traces; retry exhaustion.

### 1.4 Human-in-the-loop boundary

The system requires user approval, preference input, review, or intervention.

Examples: user approves a plan; user selects a candidate; user edits a generated artifact; user confirms a destructive action; reviewer scores an output; user provides missing requirements.

Recommendation: represent the human interaction as an explicit decision boundary, not an ad hoc pause in control flow. The boundary should expose: what decision is needed, what options are available, what context the human sees, what result shape resumes execution, and what happens on cancellation, timeout, or incomplete input.

Good tests: user accepts; user rejects; user edits; user cancels; incomplete input handled; user response resumes the workflow correctly.

## 2. Per-Layer Checklist

For each of the six harness layers: what it owns, what it should hide, friction signals, an example interface, and what to test at the boundary.

### 2.1 Context Management

**Owns:** retrieval, ranking, deduplication, compression, formatting, token budgeting, task grounding, state injection, context provenance.

**Should hide:** where context comes from, how it is ranked or compressed, how token limits are enforced, how task state is inserted into model input, how stale or conflicting context is resolved.

**Friction signals:** prompt builders directly know about memory, tool, or workflow internals; token-budget logic duplicated across callers; required grounding added ad hoc per prompt; tests assert on prompt fragments instead of context-build behavior; context differs unpredictably across similar paths; no single owner of "what reaches the model."

**Example interface:**

```python
context = context_manager.build(
    task=task,
    state=state,
    budget=context_budget,
    policy=ContextPolicy.FOCUSED,
)
```

**Good boundary tests:** given (task, state), produces deterministic context; drops low-priority context when budget is tight; preserves required grounding; separates retrieved context from task state; handles empty retrieval.

### 2.2 Tool, Skill & Adapter System

**Owns:** registration, schemas, descriptions, routing, argument validation, permission checks, invocation, output normalization, error mapping, trace events.

**Should hide:** API-specific response shapes, authentication, rate limits, tool-specific exception types, permission logic, raw transport, tool-specific retry quirks.

**Friction signals:** tool schemas duplicated in prompts and code; raw outputs reach planners or business logic; permission checks scattered across call sites; every tool handles errors differently; planners know transport-level details; no single owner of "what the model can act on."

**Example interface:**

```python
result = tool_runtime.call(
    name="search_docs",
    args=SearchDocsArgs(query=query),
    context=execution_context,
)
```

**Good boundary tests:** rejects invalid args; blocks unauthorized calls; normalizes output; maps failure into a structured error; runs against fake tools; emits trace events.

### 2.3 Execution Orchestration

**Owns:** decomposition, sequencing, branching, routing, retry/fallback decisions, stop rules, handoff rules, execution lifecycle.

**Should hide:** internal graph traversal, local branching detail, retry loop mechanics, step scheduling, intermediate bookkeeping, planner/executor coordination.

**Friction signals:** planner and executor entangled (planner directly calls tools, executor changes plan semantics); branching only encoded in prompts; stop conditions implicit; handoffs ad hoc; multi-step behavior cannot be tested without running live; no single owner of "how work proceeds."

**Example interface:**

```python
run = executor.execute(
    workflow=workflow,
    initial_state=state,
    inputs=user_request,
)
```

**Good boundary tests:** executes simple workflow; branches on intermediate result; stops when stop condition met; hands off when required; retries per policy; produces a coherent execution trace.

### 2.4 State, Memory & Artifacts

**Owns:** task state representation, intermediate artifacts, short-term memory, long-term memory, user preferences, state updates, snapshots, resume/replay data, artifact versioning.

**Should hide:** storage format, memory retrieval mechanics, preference lookup, artifact versioning details, persistence backend.

**Friction signals:** state passed as raw dicts; global mutable context affects behavior; artifact ownership unclear; state mutated as a side effect of unrelated functions; cannot snapshot, resume, or replay a run; no single owner of "what persists."

**Example interface:**

```python
state = state_store.apply(state, event=ToolResultReceived(...))
memory_context = memory.retrieve(task=task, state=state, policy=policy)
```

**Good boundary tests:** apply events to state correctly; save and restore snapshots; separate short-term from long-term memory; deterministic memory retrieval under test; migrate old artifact versions; prevent invalid transitions; replay from checkpoint.

See section 5 ("State and Artifacts") for durable-surface guidance that spans the whole harness.

### 2.5 Evaluation & Observability

**Owns:** output validation, acceptance checks, trace capture, metrics, structured logs, error attribution, root-cause support, regression fixtures.

**Should hide:** logging backend, metrics backend, evaluation implementation, trace serialization format, dashboard backend.

**Friction signals:** success judged only by final text; intermediate steps unvalidated; logs unstructured; failures cannot be attributed to a layer; no durable traces; debugging requires reconstructing what the agent saw and did; no single owner of "did the system work and why did it fail."

**Example interface:**

```python
verdict = evaluator.evaluate(
    task=task,
    trace=run.trace,
    final_output=run.output,
)
```

**Good boundary tests:** captures model calls, tool calls, and state transitions; validates final output; attributes failure to the right layer; produces replayable traces; emits expected metrics; distinguishes model failure from harness failure.

### 2.6 Constraints & Recovery

**Owns:** guardrails, policy/permission constraints, safety checks, retry policy, fallback policy, recovery logic, rollback logic, cleanup.

**Should hide:** policy evaluation details, retry mechanics, fallback routing, cleanup behavior, rollback implementation, recoverable-vs-terminal classification.

**Friction signals:** safety checks scattered; permissions checked after tool calls; retry behavior inconsistent across paths; fallback is prompt-only; failed actions leave partial state behind; no distinction between recoverable and terminal failure; no single owner of "what's allowed and how to recover."

**Example interface:**

```python
result = recovery_controller.run(
    operation=operation,
    constraints=constraints,
    recovery_policy=recovery_policy,
)
```

**Good boundary tests:** blocks disallowed action before execution; retries recoverable failure; does not retry forbidden action; falls back when primary path fails; rolls back partial state; produces clear recovery trace; cleans up after failure.

## 3. Runtime vs. Harness in Detail

The principle is in `SKILL.md`. This section adds friction signals and what a clean split looks like.

### 3.1 Task policy leaking into runtime

- A "general" workflow executor has hard-coded branches for one task family
- The runtime knows which artifact paths matter for a specific agent
- Tool adapters decide retry, escalation, or stop based on task semantics
- A generic retry loop encodes a task-specific recovery policy

### 3.2 Runtime quirks leaking into harness

- Prompts include strings shaped by tool API formats
- Controllers handle tool-specific exception types
- The harness depends on framework defaults (timeout, retry limits, sandbox flags) — flipping a config silently changes task behavior
- Test fixtures must include the full live stack to get the harness to run
- Verifier scripts implicitly drive control flow but are invisible to the harness contract

### 3.3 A clean split

- Runtime exposes generic services: call model, invoke tool, write artifact, capture trace, schedule child agent, enforce permission, retry on transport error
- Harness declares task-specific policy: this stage, this role, these gates, this recovery, this stop rule
- Adapters bridge the two — translating harness intent down into runtime calls and normalizing runtime results back into harness artifacts, events, and state updates
- Fake runtime services let harness logic be tested without live infrastructure
- Ablations (verifier on/off, repair loop, retrieval policy) require only harness changes, not runtime changes

## 4. Contracts at Three Scopes

Three nested scopes for declare-don't-imply. The principle: behavior the harness depends on should be explicit enough that callers can reason about a contract instead of inferring it from controller code.

### 4.1 Harness contract (whole task family)

What this *family* of tasks guarantees and requires. It may live in natural language, structured config, typed classes, a workflow graph, tests, or a hybrid. Form does not matter; explicitness and testability do.

A harness contract specifies:

- **Control:** stages, step order, branching rules, delegation rules, stop conditions, handoff conditions
- **Roles:** planner, solver, tool user, verifier, critic, debugger, researcher, summarizer, orchestrator (with non-overlapping responsibilities — see section 4.4)
- **Inputs:** user task, context package, state snapshot, prior artifacts, tool results, memory results, runtime config
- **Outputs:** structured objects, files, patches, reports, scores, decisions, updated state, trace events
- **Gates:** format, schema, static checks, unit tests, semantic checks, human approval, safety, permissions, budget
- **State semantics:** what persists across stages, branches, retries, child agents; what is in the manifest; what is resumable and replayable
- **Failure handling:** known failure modes, classification, retry/repair/fallback/rollback/escalation/terminal policies
- **Runtime-facing adapters:** tool, model, verifier, sandbox, filesystem, child-agent, trace adapters

Refactor signals: workflow only understandable by reading controller code; stage names implicit; failure handling unnamed; verification gates hard-coded in unrelated files; the same behavior cannot be tested without running the whole agent.

### 4.2 Stage contract (one stage in the harness)

What one stage consumes, produces, checks, and permits.

```python
StageContract(
    name="repair",
    role="debugger",
    purpose="Fix the implementation using verifier feedback",
    inputs=[ArtifactRef("solution.py"), ArtifactRef("test_result.json")],
    required_outputs=[ArtifactSpec(path="solution.py", kind="python_source")],
    allowed_tools=["read_file", "edit_file", "run_tests"],
    gates=[FormatGate("valid_python"), ScriptGate("pytest tests/")],
    retry_budget=3,
    on_failure={
        "invalid_format": "retry",
        "test_failure": "retry",
        "tool_error": "retry_once",
        "permission_violation": "stop",
        "budget_exceeded": "escalate",
    },
    on_success="stop_or_continue",
)
```

A good stage contract specifies: name, role, purpose, inputs, required outputs, optional outputs, output locations, allowed tools, permission scope, validation gates, retry budget, failure modes, recovery actions, and stop/transition rule.

Refactor signals: prompt says what to do but runtime does not enforce it; code validates outputs but does not declare required outputs; tools available globally instead of stage-scoped; retry limits buried in local control flow.

### 4.3 Agent call contract (one model or child-agent invocation)

Treat each model or child-agent call as a contract, not a raw completion.

```python
contract = AgentCallContract(
    role="Debugger",
    inputs={
        "failing_code": ArtifactRef("solution.py"),
        "test_error": ArtifactRef("test_result.txt"),
    },
    required_outputs=[RequiredArtifact(path="solution.py", kind="python_code")],
    permissions=PermissionScope(tools=["edit_file", "run_tests"]),
    budgets=Budget(max_attempts=3, max_tool_calls=10),
    gates=[FormatGate("valid_python"), ScriptGate("pytest tests/")],
    on_failure={
        "format_error": "repair",
        "test_failure": "retry",
        "permission_violation": "stop",
    },
)
result = agent_runtime.call(contract)
```

Refactor signals: code calls the model and hopes the output is usable; required artifacts described only in prompts; output paths implicit; callers parse different shapes from the same role; permissions and budgets not attached to the call; completion determined by ad hoc string checks.

Good tests at this scope: rejects result when required artifact is missing; rejects invalid format; enforces permission scope; enforces retry budget; routes failure per taxonomy; records inputs, outputs, and gates in the trace.

### 4.4 Role boundaries

A role boundary defines what one agent role is responsible for and what it must not do.

Common roles:

- **Orchestrator** — chooses next stage, routes artifacts, enforces stop and handoff
- **Planner** — decomposes work, produces a plan artifact
- **Solver** — produces the main work product
- **Tool user** — invokes tools under a bounded contract
- **Verifier** — evaluates artifacts against gates without mutating them, unless explicitly authorized
- **Critic** — identifies weaknesses or risks
- **Debugger / Repairer** — modifies artifacts based on failure feedback
- **Summarizer** — compresses history into durable state
- **Researcher** — gathers evidence and records sources

Refactor signals (role-boundary collapse): one role plans, executes, verifies, and repairs in the same opaque call; a verifier silently edits the artifact it is supposed to judge; a solver changes requirements instead of satisfying them; a planner directly mutates implementation artifacts; a child agent has vague responsibilities and returns unstructured prose; role prompts overlap so much that failures cannot be attributed.

Better boundary: each role has a contract; declared inputs and outputs; scoped tools and permissions; verification and mutation separated unless explicitly combined; delegated workers return durable artifacts, not just prose.

## 5. State and Artifacts

State semantics define what must persist across steps, branches, retries, and delegated workers — and what surface that state lives on.

### 5.1 What to make explicit

- What is transient model context vs. durable task state
- What is an artifact, decision record, trace event, or memory entry
- What is visible to or writable by child agents
- What survives retry, branching, context compaction, and restart

### 5.2 Durable state surfaces

Long-horizon agents should not keep critical state only in transient model context. State should be:

- **Externalized** — written to artifacts (`task_state.json`, `plan.md`, `verification_result.json`, `decision_ledger.md`, `child_agent_report.md`), not held only in chat history or local variables.
- **Path-addressable** — later stages reopen the exact object by path or artifact ID. Bad: "use the thing we discussed earlier." Good: "read `artifacts/plan.md` and `artifacts/test_result.json`."
- **Manifested** — produced artifacts listed in a manifest with path, producing stage and role, type, creation time, validation status, provenance, and mutability.
- **Compaction-stable** — survives context truncation, restart, branching, delegation, summarization, handoff between roles.
- **Replayable** — state changes can be reconstructed from snapshots or events.

### 5.3 Refactor signals

- Important state exists only in conversation history
- A stage depends on an artifact but does not know its path
- Child-agent outputs are copied into chat but not saved as durable objects
- No manifest of produced artifacts
- State is mutated as a side effect of unrelated functions, with no event or trace
- Branches overwrite each other's artifacts
- Retry behavior depends on hidden mutable state
- Context compaction can erase decisions or evidence

### 5.4 Better boundary

```python
artifact = artifact_store.write(
    path="artifacts/plan.md",
    content=plan,
    role="planner_output",
)
state = state_store.record(event=ArtifactProduced(stage="plan", artifact=artifact))
```

Good tests: writes required state artifacts; reopens artifacts by path; maintains a manifest; survives context compaction in replay; preserves child-agent outputs as durable artifacts; fails clearly when a required artifact is missing.

## 6. Failure Modes

### 6.1 Failure taxonomy and recovery actions

Name expected failure modes; map each to an explicit recovery action.

Common failure modes: `missing_artifact`, `wrong_path`, `invalid_format`, `schema_error`, `verifier_failure`, `test_failure`, `tool_error`, `timeout`, `permission_violation`, `budget_exceeded`, `context_missing`, `state_conflict`, `unknown_error`.

Recovery actions: `retry`, `repair`, `regenerate`, `fallback`, `escalate`, `rollback`, `stop`.

```python
failure_policy = {
    "missing_artifact": RecoveryAction.REGENERATE,
    "invalid_format": RecoveryAction.REPAIR,
    "verifier_failure": RecoveryAction.REPAIR,
    "tool_error": RecoveryAction.RETRY_ONCE,
    "permission_violation": RecoveryAction.STOP,
    "budget_exceeded": RecoveryAction.ESCALATE,
}
```

Refactor signals: all failures become generic exceptions; retry logic does not know why a step failed; recovery encoded in scattered catch blocks with no central policy; system retries terminal failures or stops on recoverable ones.

Good tests: each known failure maps to a recovery action; unknown failures stop or escalate safely; terminal failures are not retried; recoverable failures produce clear repair context; failure classification recorded in the trace.

### 6.2 Common patterns to design against

When proposing a deepening refactor, name which of these the boundary reduces.

#### Schema drift

Model or tool outputs slowly diverge from what callers expect.

Signals: raw dictionaries passed everywhere; repeated field lookups; prompt schema and code schema differ; many defensive missing-field checks.

Better boundary: typed artifacts; versioned schemas; central parser/validator; schema compatibility tests.

#### Prompt-runtime mismatch

Prompt says one thing; runtime expects another.

Signals: prompt describes fields not parsed by code; runtime expects fields not in the prompt; repair prompt has a different format than the original; prompt templates change without parser updates.

Better boundary: prompt, schema, parser, validator, and repair policy co-owned by one module.

#### Hidden state coupling

Behavior depends on implicit mutable context.

Signals: global context objects; tools read hidden session state; tests require large unrelated setup; behavior changes based on previous unrelated calls.

Better boundary: explicit state object; controlled transitions; snapshot/replay support; event-sourced or transition-based updates.

#### Retry sprawl

Retry and fallback logic appears at many call sites with no single owner.

Signals: repeated try/except around model or tool calls; copied repair prompts; inconsistent retry limits; unsafe actions retried; recoverable errors not retried.

Better boundary: central retry/repair policy; error classification; observable retry traces; explicit recoverable-vs-terminal distinction.

#### Tool leakage

Tool-specific details leak into planning or business logic.

Signals: API response shapes used directly by planners; auth or rate-limit handling spread across code; tool exceptions handled inconsistently; planner knows transport details.

Better boundary: tool port; production adapter; test adapter; normalized result and error artifacts.

#### Non-replayable execution

Failures are hard to reproduce.

Signals: logs are unstructured; tool/model calls not recorded; state snapshots missing; tests cannot replay real failures; cannot reconstruct what the model saw.

Better boundary: trace object; event log; deterministic replay runner; captured artifacts; captured model inputs and outputs.

#### Weak failure attribution

The system knows that a task failed, but not why.

Signals: final error is generic; no layer-level error categories; no trace linking output to model/tool/state events; debugging requires manual reconstruction.

Better boundary: observability layer that attributes failure to context, model, tool, state, constraints, orchestration, or evaluation.

## 7. Test Strategy

The principle: replace shallow tests with boundary tests. Once a deep module exists, tests should move to its boundary.

### 7.1 Prefer

- **Boundary tests** — exercise the public interface of the deep module.
- **Contract tests** — verify adapters obey the same port contract.
- **Trajectory tests** — verify a representative multi-step agent path.
- **Replay tests** — replay a saved synthetic or real trace.
- **Failure-injection tests** — inject malformed model output, tool failure, timeout, invalid state, partial execution, retry exhaustion.
- **Schema-evolution tests** — verify old artifacts can still be read or migrated.
- **Observability tests** — verify trace events, metrics, and failure attribution.
- **Recovery tests** — verify retry, fallback, rollback, cleanup, and terminal failure behavior.
- **Harness ablation tests** — compare runs with and without one harness module or policy. Useful for attributing improvement to context, verification, repair loops, state semantics, recovery policy, or model choice rather than to whole-controller changes.

### 7.2 Avoid over-investing in

- Tests for tiny prompt-string helpers after a context or validation boundary exists
- Tests for private parsing helpers after a boundary parser exists
- Tests that assert incidental internal call order
- Tests that require live hosted/network services unless explicitly marked integration/E2E
- Tests that inspect internal mutable state instead of observable behavior
- Tests that hard-code incidental prompt wording
- Tests that duplicate the implementation instead of testing the contract

### 7.3 Examples

```python
# Boundary test
result = runtime.run(name, input, context)
assert result.status == "success"

# Contract test
assert_contract(SearchToolPort, InMemorySearchTool())
assert_contract(SearchToolPort, ProductionSearchTool(...))

# Trajectory test
trace = agent.run(task)
assert trace.completed()
assert trace.used_tool("code_search")

# Replay test
replay = replay_runner.replay("fixtures/traces/failed_parse_then_repair.json")
assert replay.final_status == "success"

# Failure injection
model.enqueue_response("{bad json")
model.enqueue_response(valid_json)
result = planner.plan(task)
assert result.status == "success" and result.repair_attempts == 1

# Ablation
baseline = run_harness(task, harness.without("repair_loop"))
candidate = run_harness(task, harness.with_module("repair_loop"))
assert candidate.success_rate >= baseline.success_rate
```

## 8. RFC Template

Use this template when producing a refactor RFC at step 7 of the process. Fill in only the sections the candidate has substance for; do not pad.

---

### Problem

Describe the architectural friction:

- The harness responsibility currently split across files, prompts, schemas, adapters, or tests with no clear contract surface or single owner
- Concrete evidence — file paths, call sites, tests, traces, artifacts
- Which agent failure modes this creates (see section 6)
- Why this makes the system harder to test, debug, replay, recover, or extend

### Proposed Interface

- Interface signature: types, methods, params, return artifacts
- Usage example from a likely caller
- Complexity hidden internally
- Invariants the interface enforces
- Artifacts owned or produced
- What remains outside the boundary

### Boundary Strategy

State the boundary category (section 1) and how dependencies are handled:

- **In-process agent logic** — merged behind a deeper interface
- **Local-substitutable** — tested with a local stand-in
- **LLM reasoning** — prompt/schema/parser/validator/repair/fallback/trace policy
- **Human-in-the-loop** — explicit decision object and resume result

Describe what the boundary owns across the affected harness layer(s): context, tool runtime, execution orchestration, state/memory/artifacts, evaluation/observability, or constraints/recovery. Describe state and artifact flow, error model, and trace/replay model. Note the runtime-vs-harness split: what belongs to generic runtime services vs. task-family policy.

### Testing Strategy

- New boundary, contract, trajectory, replay, failure-injection, observability, and recovery tests
- Old shallow tests to delete or rewrite
- Test environment needs: fake models, local stand-ins, deterministic clocks, temporary workspaces, in-memory adapters, replay fixtures

### Migration Plan

- How callers move to the new interface
- Compatibility or adapter shims needed during migration
- Risks and rollback plan
- Milestones that keep the system working after each step
