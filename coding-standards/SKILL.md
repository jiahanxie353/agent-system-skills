---
name: coding-standards
description: Coding conventions for naming, readability, immutability, and code-quality review.
---

# Coding Standards & Best Practices

Coding conventions applicable across projects.

## When to Activate

- Starting a new project or module
- Reviewing code for quality and maintainability
- Refactoring existing code to follow conventions
- Enforcing naming, formatting, or structural consistency
- Setting up linting, formatting, or type-checking rules
- Onboarding new contributors to coding conventions

## Scope Boundaries

Activate this skill for:
- descriptive naming
- immutability defaults
- readability, KISS, DRY, and YAGNI enforcement
- error-handling expectations and code-smell review
- general code-quality review across the agent system

Do not use this skill as the primary source for:
- agent-system design decisions
- domain-specific framework guidance when a narrower rule already exists

## Code Quality Principles

### 1. Readability First

- Code is read more than written
- Clear variable and function names
- Self-documenting code preferred over comments
- Consistent formatting

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

## Naming Conventions

Use `naming_convention.md` as the detailed source of truth for Python naming rules.

## Comments & Documentation

Follow `comments_docstrings.md` for Python comments and docstring rules.

## Python Patterns

Follow `python_patterns.md` for Python-specific implementation patterns, including type hints, data modeling, immutability, error handling, imports, comprehensions, context managers, package organization, and common anti-patterns.

## Remember

Code quality is not negotiable. Clear, maintainable code enables rapid development and confident refactoring.
