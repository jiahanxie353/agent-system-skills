---
name: git-commit-helper
description: Generate clear conventional commit messages for changes to an agent-system codebase. Use when staging changes, asking for a commit message, or before committing to repos with prompts, tools, evals, runtime orchestration, memory, or traces. Adds agent-specific commit types such as prompt, tool, contract, runtime, state, eval, trace, safety, and model, and surfaces coupled changes that must land together.
allowed-tools: Bash, Read
---

# Agent-System Commit Helper

Generate conventional commit messages from staged changes in an agent-system codebase.

The bar is the same as anywhere else: each commit names *what behavior changed* and *why*, in imperative mood, with a body when needed. Two things shift in an agent-system codebase, and this skill captures both:

1. **The type vocabulary widens.** Standard types (`feat`, `fix`, `refactor`, ...) still apply, but a `feat(planner)` does not tell future readers whether the diff was a prompt rewrite, a tool registration, a contract change, a parser update, or a runtime policy change. Agent-system additions (`prompt`, `tool`, `contract`, `runtime`, `state`, `eval`, `trace`, `safety`, `model`) name the artifact directly.
2. **Atomicity is stricter.** A prompt-output change must land with its parser. A tool-schema change must land with the caller code that branches on the shape. A state-shape change must land with migration or replay-fixture updates. The type checker will not catch contract drift across two commits.

See [Conventional Commits](https://www.conventionalcommits.org/) for the underlying conventional-commits foundation.

## When to use

- User runs `git commit` without a message
- Staged changes exist and the user wants a message generated
- Changes touch prompts, tools, evals, runtime orchestration, memory/context, or telemetry
- User asks to clean up or rewrite a vague message
- User wants to decide whether staged changes should be split into multiple commits

## Format

```text
<type>[optional scope][!]: <subject>

[optional body]

[optional footer(s)]
```

**Subject** — imperative mood ("add", not "added" or "adds"), lowercase, no trailing period, ≤ 50 chars when possible. Names the *behavior or contract* that changed, not the file edited. The subject should complete the sentence "If applied, this commit will ___".

Good:

- `prompt(planner): require stop_reason on every step`
- `tool(search): normalize empty results to []`
- `fix(executor): resolve race in async tool dispatch`
- `refactor(retry): consolidate retry policy into one module`

Bad:

- `added user authentication` — past tense
- `fix: bug fix` — vague
- `Update API docs.` — capitalized and has trailing period
- `feat: stuff` — says nothing
- `update planner.txt` — names the file edit, not the behavior

**Body** — wrap at 72 chars, separated from the subject by a blank line. Use bullets when listing coordinated changes. Cover:

- *Why* the change was made: trace, eval, incident, issue, or design motivation
- What contract or behavior moved
- Which coupled artifacts moved with it
- Caller-visible impact
- Evaluation, latency, cost, safety, or reproducibility impact when relevant

Explain the *why*, not the *how*. The diff shows how.

**Footer** — `Closes #N`, `Fixes #N`, `Refs trace <id>`, `Refs eval <case>`, `Refs incident <date>`, or `BREAKING CHANGE: <description>` for contract changes.

## Types

Standard types still apply:

`feat`, `fix`, `refactor`, `perf`, `docs`, `test`, `style`, `chore`, `build`, `ci`, `revert`.

Agent-system additions:

| Type       | Use when the change affects                                                |
| ---------- | -------------------------------------------------------------------------- |
| `prompt`   | system instructions, planner/repair/judge prompts, few-shot examples |
| `tool`     | tool adapters, schemas, descriptions, permissions, output normalization    |
| `contract` | tool/artifact schemas, preconditions, postconditions, invariants     |
| `runtime`  | orchestration, scheduling, branching, retry, fallback, timeouts            |
| `state`    | context packing, summarization, memory retrieval, scratchpad, task-state   |
| `eval`     | benchmarks, regression traces, judge prompts, metrics, failure taxonomy    |
| `trace`    | logs, traces, telemetry, cost tracking, failure attribution                |
| `safety`   | guardrails, sandboxing, permissions, policy checks                         |
| `model`    | model selection, version pinning, decoding parameters                      |

Avoid defaulting to `feat`. In agent-system repos, the most useful type usually names the artifact that changed: `prompt`, `tool`, `contract`, `runtime`, `state`, `eval`, or `trace`.

Use `contract` when externally relied-upon meaning changes, even if no JSON schema changed. Examples include required fields, artifact invariants, preconditions, postconditions, stop conditions, retry classifications, prompt-output formats, trace semantics, and evaluator assumptions.

Pick the type that names the dominant artifact. If a diff spans two types and they are not contract-coupled, it should usually be split into two commits.

### Dominant type examples

- Prompt-output contract change + parser update → `prompt` or `contract`
- Tool schema change + caller update → `tool` or `contract`
- New eval exposing a bug + fix → split into `eval`, then `fix`
- Runtime support added only to satisfy a new artifact contract → usually `contract`
- Telemetry field rename consumed by dashboards → `trace!` or `contract!`

## Common scopes

Name the agent, capability, or module — not the directory or filename:

`planner`, `router`, `tool`, `parser`, `executor`, `runtime`, `memory`, `context`, `evaluator`, `judge`, `trace`, `sandbox`, `workflow`, `repair`, `benchmark`

A bare type with no scope is fine when the change is genuinely cross-cutting:

```text
chore: bump anthropic SDK to 0.34
```

Avoid:

- `fix(src)`
- `feat(planner.py)`
- `chore(files)`

Directories and filenames are not responsibilities.

## Coupling: keep together vs split

Land **in one commit** when splitting creates a broken intermediate state. When you keep coupled changes together, list the coupled artifacts in the body so future bisection can find the link.

## Split decision

Before writing the message, decide:

1. Does the diff contain more than one logical behavior change?
   - If no, write one commit.
   - If yes, continue.

2. Would splitting leave either commit broken because of a changed prompt, tool, schema, state, artifact, or trace contract?
   - If yes, keep together and list coupled artifacts.
   - If no, split.

3. Is there a new eval and a fix?
   - Prefer two commits: failing eval first, fix second.

4. Is part of the diff a mechanical refactor?
   - Split the refactor from behavior changes unless the behavior change cannot be made safely without it.

5. Are generated traces, fixtures, or snapshots included?
   - Name the upstream behavior that forced regeneration in the subject or body.
   - Do not make fixture refresh the only visible explanation.

## Breaking changes

In an agent-system codebase, "breaking" includes contract changes the type checker will not catch. Mark with `!` and a `BREAKING CHANGE:` footer for:

- Tool input or output schema changes
- Prompt-output contract changes: new required fields, renamed fields, removed fields
- State or artifact shape changes that need migration
- Trace format changes consumed by downstream tooling
- Retry/fallback classification changes callers depend on
- Model pin changes that move a stable eval baseline
- Eval rubric changes that invalidate previous pass-rate comparisons

Conventional placement: put the explanatory body first, then the `BREAKING CHANGE:` footer last.

## Analysis process

### Step 1: Inspect staged changes

```bash
git diff --staged --name-only
git diff --staged
```

Optionally inspect recent commit style:

```bash
git log --oneline -n 20
```

### Step 2: Categorize by artifact

For each changed file, ask what kind of artifact it is.

| File kind                                   | Likely type             |
| ------------------------------------------- | ----------------------- |
| Prompt template, system instruction         | `prompt`                |
| Tool adapter, tool description              | `tool`                  |
| Schema, precondition, postcondition spec    | `contract`              |
| Orchestration / scheduling / retry code     | `runtime`               |
| Memory, scratchpad, context-packing logic   | `state`                 |
| Eval fixture, golden trace, judge prompt    | `eval`                  |
| Logs, traces, telemetry                     | `trace`                 |
| Guardrails, sandboxing, permissions         | `safety`                |
| Model config, version pin, decoding params  | `model`                 |
| Test files                                  | `test`                  |
| Documentation                               | `docs`                  |
| Build, CI, dependencies                     | `build`, `ci`, `chore`  |
| New regular code                            | `feat`                  |
| Modified regular code, behavior fix         | `fix`                   |
| Modified regular code, no behavior change   | `refactor`              |

If files span multiple kinds, decide whether they are **coupled by contract** or **accidentally batched**.

### Step 3: Analyze content

Ask:

- What behavior or contract changed?
- Why was it changed? Trace? Eval? Incident? Issue?
- Does it alter tool schemas, prompt-output contracts, or artifact formats?
- Does it affect retry, fallback, recovery, or stop-condition classification?
- Does it move an eval baseline or cost/latency/safety metric?
- Could existing callers, trajectories, or dashboards break?
- Are generated traces or fixtures refreshed? If so, what upstream behavior forced that refresh?

### Step 4: Write the message

Pick the type and scope that name the dominant responsibility. Write the subject as the *behavior or contract change*, not the file edit. If non-trivial, add a body covering motivation, contract change, coupled artifacts, and caller impact. Reference traces, evals, issues, or incidents in the footer. Mark breaking changes with `!` and a `BREAKING CHANGE:` footer.

## Examples

### Contract change

```text
contract(tool)!: require explicit preconditions in tool specs

- Add precondition and postcondition sections to tool metadata
- Validate required fields before runtime execution
- Surface contract violations as structured repair feedback

BREAKING CHANGE: existing tools without contract fields now fail
validation.
```

### Runtime fallback

```text
runtime(planner): add fallback path for failed tool calls

- Route schema errors to the repair prompt instead of aborting
- Preserve failed call arguments in task state
- Retry with normalized tool output after parser recovery

Refs trace 2026-04-21-7f3a
```

### Coupled prompt + parser change

```text
prompt(planner): require explicit stop_reason on every step

Decoder previously accepted steps without stop_reason and defaulted
to "continue", masking silent infinite-loop failures observed in
trace 2026-04-21-7f3a.

Coupled changes:
- prompt(planner): add stop_reason to step contract
- runtime(executor): treat missing stop_reason as terminal
- eval(planner-decomposition): regenerate goldens

Refs trace 2026-04-21-7f3a
```

### Tool schema breaking change

```text
tool(search)!: return [] instead of null on empty results

Empty-result branches were a recurring source of attribution ambiguity:
was it tool failure or an empty result? Normalizing at the adapter
removes the ambiguity.

Coupled changes:
- tool(search): adapter normalizes null to []
- prompt(researcher): drop "results are null" branch
- eval(researcher-suite): refresh fixtures

BREAKING CHANGE: tool(search) output contract no longer permits null.
Callers that branched on `result is None` must check `len(result) == 0`.

Closes #214
```

### Evaluation update

```text
eval(agent): add regression traces for routing failures

- Capture successful and failed tool-selection trajectories
- Add judge rubric for whether the selected tool was appropriate
- Track false-positive tool activations across benchmark tasks

Refs incident 2026-04-19
```

### Context management

```text
state(context): compact tool observations before replanning

- Summarize long tool outputs into structured observations
- Preserve command, status, artifact path, and failure reason
- Reduce planner context size without dropping execution state
```

### Prompt change

```text
prompt(repair): tighten patch-format instructions

- Require exact SEARCH/REPLACE markers
- Add examples for malformed separator recovery
- Explain how syntax errors should be reflected back to the agent
```

### Refactor with no behavior change

```text
refactor(retry): consolidate retry policy into one module

Retry behavior was split across executor.run, tool_runner.invoke, and
three caller sites. Single owner makes terminal-vs-recoverable
classification testable in isolation.

No behavior change. New unit tests cover rate-limit, schema-error, and
timeout classifications. Trajectory suite passes unchanged.
```

### Model pin

```text
model(planner): pin to claude-opus-4-7

Floating reference produced silent drift between runs of the
planner-decomposition eval. Pinning stabilizes the baseline; future
bumps are explicit commits.
```

### Trace / observability

```text
trace(executor): split tool_failed into schema and runtime errors

Failure events previously rolled tool-call errors into one
"tool_failed" category, defeating attribution. Split into
"tool_schema_invalid" and "tool_runtime_error" so the dashboard
distinguishes contract bugs from transient tool failures.
```

### Generated trace refresh

```text
eval(router): refresh traces for no-result search contract

- Regenerate router goldens after search now returns [] on empty results
- Preserve previous no-tool-needed cases unchanged
- Document fixture refresh reason in the trace manifest

Refs eval router-no-result-search
```

## Anti-patterns

- **`update prompt`** — invisible to bisection; does not name the contract change
- **`fix bug` for trace-derived fixes** — name the failure mode: parse failure, schema mismatch, retry exhaustion, invalid stop condition
- **Silent fixture refresh** — regenerating a golden trace without naming the upstream cause hides the link between behavior and fixture
- **Schema change without caller update** — splits a contract across commits and produces a broken intermediate state
- **Floating model references** — bumping a model without pinning makes the commit unreproducible
- **Reclassifying retry policy silently** — flag it; even though no signature changes, callers depend on it
- **Overusing `feat`** — agent-system changes are often better described as `prompt`, `tool`, `runtime`, `state`, `eval`, or `contract`
- **Naming files instead of behavior** — `update planner.py` does not say what changed
- **Mixing refactor and behavior change** — makes regression bisection harder

## Tips

### General

1. **Be specific.** Name the behavior, not the file.
   - Good: `fix(executor): resolve race in async tool dispatch`
   - Bad: `fix: update file`

2. **Use imperative mood.** "Add", not "added" or "adds". The subject should complete "If applied, this commit will ___".

3. **Explain *why*, not *how*.** The diff shows how. The commit message captures the background and intent.

4. **Make commits atomic.** One logical change per commit. Each commit should leave the code in a working state, so bisection finds the right cause.

5. **Reference issues and motivating context.** `Closes #N`, `Fixes #N`, `Refs trace <id>` — link to the failure mode that motivated the change.

6. **Flag breaking changes explicitly.** Use `!` after type and `BREAKING CHANGE:` in the footer.

7. **Match recent history.** Skim `git log` before writing and match the type, scope, and tone the project already uses.

### Agent-system specific

8. **Prefer agent behavior over implementation detail.**
   - Good: `runtime(planner): retry malformed tool calls`
   - Bad: `fix: update python file`

9. **Mention contracts explicitly.** If a schema, artifact, or invariant moved, say so.

10. **Separate prompt and runtime changes** when they are not contract-coupled.

11. **Explain evaluation impact.** If a change affects pass rate, cost, latency, reproducibility, or safety, include it in the body.

12. **Reference traces, evals, and incidents** in the footer. Agent bugs often surface from trajectories, not stack traces.

13. **Treat generated artifacts as evidence, not the headline.** Name the behavior that forced the trace, fixture, or snapshot refresh.

14. **Preserve auditability.** Future readers should be able to answer: what agent behavior changed, why did it change, what contract moved, and what evidence motivated it?
