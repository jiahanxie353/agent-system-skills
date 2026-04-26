---
name: improve-agent-system-architecture
description: Explore an agentic system codebase to find architectural refactoring opportunities in the harness around models, tools, workflows, state, memory, evaluation, and recovery. Use when the user wants to deepen harness modules, refactor agent execution paths, simplify orchestration glue, or make agent behavior easier to test, replay, attribute, and reason about.
---

# Improve Agent System Architecture

Explore an agentic system codebase, surface architectural friction, and propose refactors that turn scattered orchestration glue into **deep harness modules**: small, stable interfaces that hide the complexity of multi-step reasoning, tool use, memory, validation, and recovery.

In an agent system, reliability mostly comes from the harness around the model, not from any single prompt or model call. The harness is the most important thing — a good agentic-system architecture makes its six layers explicit: **context management** (what reaches the model), **tool system** (what the model can act on), **execution orchestration** (how work proceeds step by step), **state and memory** (what persists during and across tasks), **evaluation and observability** (whether the system works and why it failed), and **constraints and recovery** (what is allowed and how to recover when things break).

The refactor target is the *effective harness*: the hidden control stack that governs how work is decomposed, what state persists, which tools run when, what gates an artifact must pass, and how failures are classified and recovered.

A strong candidate has this shape: one harness responsibility, such as building the model's context, choosing retry and recovery policy, validating tool outputs, or advancing workflow state, is split across code, prompts, state, tests, or adapters without a clear contract surface or single owner.

File spread is not itself the smell. The smell is **split ownership**: changing one behavior requires coordinated edits across files, prompts, schemas, adapters, or tests because no explicit boundary owns the behavior.

The refactor gives that responsibility one deep boundary: a small interface that owns the contract, hides orchestration complexity, and lets callers reason about the behavior instead of coordinating implicit edits across modules.

A useful sanity check: would another AI agent reading this codebase understand the harness from explicit boundaries, or would it have to infer behavior from controller code, framework defaults, prompt fragments, and tool adapter quirks? If the latter, there is a refactor.

## Harness Layers

Use as lenses, not a rigid taxonomy — a strong refactor candidate may span more than one.

1. **Context Management** — retrieval, ranking, compression, formatting, token budget, task grounding, provenance.
2. **Tool, Skill & Adapter System** — registration, schemas, permissions, invocation, output normalization, error mapping.
3. **Execution Orchestration** — decomposition, sequencing, branching, retries, stop and handoff rules, lifecycle.
4. **State, Memory & Artifacts** — task state, durable artifacts, manifests, short/long-term memory, snapshots, replay data.
5. **Evaluation & Observability** — output validation, gates, traces, metrics, failure attribution, regression fixtures.
6. **Constraints & Recovery** — guardrails, permission checks, retry policy, fallback, rollback, cleanup, terminal-vs-recoverable classification.

Cross-cutting principle: keep **runtime responsibilities** distinct from **harness responsibilities**, even when they live in adjacent files.

The runtime owns reusable execution machinery: model/tool adapters, sandboxing, permission enforcement mechanics, scheduling, persistence, trace capture, cancellation, and retry plumbing.

The harness owns task-family policy: stages, roles, prompts, task-specific contracts, gates, artifact ownership, state semantics, recovery policy, and stop/handoff rules.

Adapters translate harness intent into runtime calls, then normalize runtime results back into harness artifacts, events, and state updates.

Flag friction when task policy leaks into generic runtime code, runtime quirks leak into prompts/controllers, or behavior cannot be tested without the full live stack.

## Process

### 1. Explore concrete execution paths

Start from real entry points, tests, demos, traces, or docs. Trace one to three representative agent tasks end to end. Use `Agent` with `subagent_type=Explore` for breadth; otherwise inspect locally.

Collect evidence before naming candidates:

- File paths and core abstractions involved
- Call flow, state transitions, artifact flow
- Prompt / schema / parser / validator / tool / state / trace shapes
- Existing tests and what behavior they actually protect
- Places where understanding requires chasing many shallow helpers
- Places where failure cannot be attributed, replayed, or tested locally
- Places where harness policy is buried inside generic runtime code, or vice versa

The friction you encounter IS the signal.

### 2. Present refactor candidates

Present a numbered list. Each candidate must be grounded in code evidence, not generic agent-system advice.

For each candidate, show:

- **Harness responsibility** that should become explicit
- **Layer lens** — one or more of the six
- **Evidence** — file paths, call sites, tests, traces
- **Current coupling** — shared state, schema assumptions, call order, retry behavior, permission logic, artifact shape, context dependency
- **Friction** — why this is hard to understand, test, replay, debug, recover, or extend
- **Hidden assumption** — what is implicit in controller code, framework defaults, prompts, adapters, or verifier scripts
- **Boundary category** (see `REFERENCE.md`)
- **Failure modes reduced** (see `REFERENCE.md`)
- **Test impact** — boundary, contract, trajectory, replay, failure-injection, observability, or recovery tests to add; shallow tests that may become redundant

Do NOT propose interfaces yet. Ask the user which candidate to explore.

### 3. Frame the selected problem

Before designing, write a user-facing framing of the chosen candidate. Include:

- The current friction and the harness responsibility being deepened
- The behavior the new boundary must preserve
- The state, artifacts, prompts, validators, tools, adapters, or policies it must coordinate
- The contracts, gates, permissions, and runtime-facing adapters it must clarify
- The failures it should prevent, detect, attribute, or recover from
- A rough illustrative code sketch to make the constraints concrete

The sketch is NOT the proposal. It only grounds the constraints. Show this to the user, then immediately proceed to step 4 — the user can read while sub-agents work.

### 4. Design multiple interfaces in parallel

Spawn 3+ sub-agents via `Agent`. Each must produce a materially different interface for the deepened module.

Apply different design pressures (pick 3-5):

- **Minimal boundary** — 1-3 public entry points
- **Explicit runtime model** — first-class state transitions, artifacts, phases, trace events
- **Common path optimized** — trivial default caller path; advanced behavior configurable
- **Ports and adapters** — external tools, models, services isolated behind ports
- **Replay/debug optimized** — deterministic replay, failure attribution, trace inspection are primary

Give each sub-agent a self-contained brief: relevant file paths, the harness responsibility, current execution flow, current artifact and state shapes, current validation and recovery behavior, observability gaps, boundary category, failure modes to reduce, what complexity should hide.

Each sub-agent outputs:

1. Interface signature: types, methods, params, return artifacts
2. Usage example from a likely caller
3. Responsibility owned and complexity hidden
4. Contract and gate strategy: inputs, required outputs, validation, permissions, budgets, stop conditions
5. State and artifact strategy: durable artifacts, manifests, path-addressable objects, snapshots, replay
6. Boundary strategy: how models, tools, services, humans, validators are handled
7. Testing, observability, and recovery strategy
8. Trade-offs

### 5. Compare and recommend

Present designs sequentially, then compare in prose. Be opinionated — the user wants a strong read, not a menu. If a hybrid is strongest, propose the hybrid explicitly.

### 6. User picks

The user picks one design or accepts your recommendation.

### 7. Produce an RFC

Default: produce a refactor RFC using the template in `REFERENCE.md`.

Do NOT create a GitHub issue, make network calls, or implement the refactor unless the user asks. If they ask, use the RFC as the issue body and share the URL.

## Reference

Read `REFERENCE.md` when you need:

- Boundary categories for agent systems (in-process, local-substitutable, remote-but-owned, true-external, LLM reasoning, human-in-the-loop)
- Per-layer checklist: friction signals, what each layer should hide, good boundary tests
- Runtime-vs-harness distinction
- Contract patterns: harness contract, stage contract, agent-call contract
- Durable state surfaces and state semantics
- Failure-mode taxonomy and recovery actions
- Common refactor targets per layer with example interfaces
- Test strategy: boundary, contract, trajectory, replay, failure-injection, schema-evolution, observability, recovery, ablation
- RFC template
