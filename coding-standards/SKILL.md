---
name: coding-standards
description: Language-neutral coding conventions for naming, readability, immutability, testing, and code-quality review.
---

# Coding Standards & Best Practices

Coding conventions applicable across projects and languages. Use this skill for shared code-quality expectations first, then load the relevant language-specific reference when the task involves a concrete implementation language.

## When to Activate

- Starting a new project or module
- Reviewing code for quality and maintainability
- Refactoring existing code to follow conventions
- Enforcing naming, formatting, or structural consistency
- Setting up linting, formatting, or type-checking rules
- Onboarding new contributors to coding conventions
- Working across multiple implementation languages

## Scope Boundaries

Activate this skill for:
- descriptive naming
- immutability defaults
- readability, KISS, DRY, and YAGNI enforcement
- error-handling expectations and code-smell review
- testing expectations and maintainability review
- general code-quality review across the agent system

Do not use this skill as the primary source for:
- agent-system design decisions
- domain-specific framework guidance when a narrower rule already exists
- language details when a language-specific reference in this skill gives a more precise rule

## How to Apply

1. Start from the shared principles in this file.
2. Identify the language or languages in scope.
3. Load the matching language-specific reference files before giving detailed implementation or review guidance.
4. If language-specific rules conflict with generic preferences, follow the idioms and safety model of the implementation language.

## Language References

### Python

- `python/naming_convention.md` for Python naming rules.
- `python/comments_docstrings.md` for Python comments and docstring rules.
- `python/python_patterns.md` for Python implementation patterns, including type hints, data modeling, immutability, error handling, imports, comprehensions, context managers, package organization, and common anti-patterns.

### Rust

- `rust/patterns.md` for Rust ownership, error handling, traits, generics, concurrency, crate structure, and idiomatic implementation patterns.
- `rust/testing.md` for Rust unit tests, integration tests, async testing, property-based testing, mocking, coverage, and TDD workflow.

## Shared Principles

### 1. Readability First

- Code is read more than written
- Clear variable, function, type, and module names
- No magic numbers: use named constants for non-obvious numbers and limits
- Self-documenting code preferred over comments
- Consistent formatting and organization

### 2. KISS (Keep It Simple, Stupid)

- Simplest solution that works
- Avoid over-engineering
- No premature optimization
- Easy to understand > clever code

### 3. DRY (Don't Repeat Yourself)

- Extract common logic into functions
- Extract reusable functions, classes, or modules when duplication represents the same concept
- Share utilities across modules
- Avoid copy-paste programming

### 4. YAGNI (You Aren't Gonna Need It)

- Don't build features before they're needed
- Avoid speculative generality
- Add complexity only when required
- Start simple, refactor when needed

### 5. Explicit Error Handling

- Make failure modes visible at API boundaries
- Preserve context when propagating errors
- Avoid silent fallbacks unless the behavior is deliberate and documented
- Do not hide correctness failures behind broad exception or error swallowing

### 6. Immutability by Default

- Prefer immutable values and narrow mutation scopes
- Make ownership and lifecycle obvious
- Avoid shared mutable state unless it is necessary and controlled
- Keep side effects localized

### 7. Testable Design

- Keep behavior easy to exercise with focused tests
- Test public behavior and important edge cases
- Use dependency boundaries that allow isolation without excessive mocking
- Add tests proportional to risk and blast radius

## Remember

Code quality is not negotiable. Clear, maintainable code enables rapid development and confident refactoring.
