# Python Coding Standard for AI Agents (py-guide)

A robust, production-grade Python coding standard and rule set specifically engineered for AI coding agents, automated workflows, and pair-programming assistants.

---

## Overview

Automated code generation tools frequently suffer from predictable anti-patterns: swapping dependencies without justification, introducing race conditions through naive file checks, executing I/O at import time, or swallowing exceptions silently.

The `py-guide` skill provides a comprehensive, prioritized rule set that enforces high standards of correctness, security, architectural discipline, and style across both existing repositories and greenfield Python projects.

---

## Rule Precedence Hierarchy

When conflicting priorities arise during code generation or refactoring, decisions must follow this strict order of precedence:

1. **Inspect Project**: Respect existing project setup, tooling, module layout, and dependencies.
2. **Correctness**: Ensure logic is sound, handles edge cases, and satisfies functional requirements.
3. **Security**: Guarantee zero hardcoded secrets, safe input validation, and secure process execution.
4. **Robustness**: Enforce resilient error handling, concurrency safety, and proper resource lifecycles.
5. **Performance**: Optimize only when supported by profiling, scale requirements, or measured latency targets.
6. **Style**: Apply consistent formatting, clear naming, and structural simplicity.

A fast or stylish implementation that is incorrect or insecure is strictly unacceptable.

---

## Core Principles and Guidelines

### 1. Project Inspection Before Action
- Inspect Python version, linter/formatter (e.g., Ruff), type checker (mypy/pyright), and test framework before making changes.
- Match existing conventions rather than forcing unsolicited rewrites or modernizations (e.g., do not replace `pandas` with `polars` or `json` with `orjson` on an existing codebase).
- Greenfield defaults: `uv` for package management, `Ruff` for linting/formatting, `mypy`/`pyright` for typing, `pytest` for testing.

### 2. Correctness over Premature Optimization
- Write straightforward, readable, and testable code first.
- Introduce specialized performance libraries only when data volume or strict latency thresholds justify their inclusion.

### 3. O(1) Complexity for Magic Methods and Properties
- `__len__`, `__bool__`, `__contains__`, and `@property` accessors must never execute I/O operations, network requests, or costly iterations over collections.
- Any expensive operation must be exposed as an explicit method (e.g., `get_row_count()`).

### 4. Concurrency-Safe Error Handling (LBYL vs. EAFP)
- **LBYL (Look Before You Leap)**: Restricted to locally owned, in-memory state where external actors cannot mutate state between inspection and use (e.g., verifying a local dictionary key or argument type).
- **EAFP (Easier to Ask for Forgiveness than Permission)**: Mandatory for external resources (files, network connections, databases, subprocesses) to prevent Time-of-Check to Time-of-Use (TOCTOU) race conditions.
- Catch specific exceptions only; never use bare `except:` or `except Exception: pass`.
- Preserve cause chains when re-raising using `raise ... from e`.

### 5. Strict Modern Typing
- Fully type-hint all public interfaces and signatures.
- Use modern Python type syntax (`list[str]`, `dict[str, int]`, `str | None`).
- Use `Literal` and enumerations for bounded string sets.
- Avoid blind `typing.cast()` calls; verify assumptions with explicit assertions where practical.

### 6. Structural Constraints for Generated Code
- Restrict indentation to a maximum of 4 levels per function; use early returns to flatten nested blocks.
- Keep functions focused and declare variables close to their initial point of use.
- Use keyword-only parameters (`*`) once a function exceeds 4 arguments; transition to a `dataclass` if configuration grows unwieldy.
- Avoid non-obvious default parameters that can silently mask bugs.

### 7. Module Lifecycle and Clean Imports
- Use top-level, absolute imports; avoid wildcard imports and re-export abstractions.
- **Zero Import-Time I/O**: No network requests, environment variable reads, filesystem operations, or client initializations at the module root level. Defer initialization using functions or cached factories (`functools.cache`).

### 8. Resilient File and Path Operations
- Exclusively use `pathlib.Path` instead of legacy `os.path` functions.
- Always specify `encoding="utf-8"` explicitly for all text file read and write operations.
- Distinguish between path-string checks (e.g., verifying path existence before `.resolve()`) and file content access (which must rely on try/except blocks to stay concurrency-safe).

### 9. Security and Resource Management
- Never hardcode secrets, access tokens, or private credentials.
- Do not log sensitive payloads, authentication headers, or personally identifiable information (PII).
- Validate untrusted input at all application boundaries.
- Run subprocesses using argument lists with explicit timeouts, `check=True`, and captured outputs.
- Always use `with` statements for external resources to guarantee deterministic cleanup.

### 10. Verification Before Handoff
Before finalizing code changes, complete the following verification steps:
- Run linting and formatting tools.
- Run type checkers.
- Execute unit and integration test suites.
- Conduct a security review pass for injection vectors, leaked credentials, and unsafe logging.

---

## Primary Use Cases

### 1. AI Agent Instruction and System Prompts
Serves as an operational standard and reference for AI coding agents (such as Antigravity, Claude, Copilot, or Cursor) to ensure generated code conforms to enterprise-grade quality and safety standards.

### 2. Greenfield Project Initialization
Provides an immediate, production-ready set of baseline conventions for bootstrapping new Python services, CLI utilities, and backend microservices with modern tooling (`uv`, `Ruff`, `pytest`, `mypy`).

### 3. Legacy Codebase Refactoring and Maintenance
Prevents agents from breaking established patterns or introducing unwanted dependencies when modifying existing repositories.

### 4. Open Source Software (OSS) Contributions
Ensures automated contributions adhere strictly to the target repository's existing tooling, style, and dependency footprint without generating unnecessary maintenance friction.

### 5. Automated Quality Gates and Code Reviews
Functions as a structured evaluation rubric for pull request bots, code review checklists, and CI verification pipelines.

---

## Reconciliation with Industry Standards

The guidelines reconcile standard Python idioms with industry agent benchmarks (such as Dagster's *Dignified Python* agent specifications):
- LBYL is accepted and recommended for internally managed, deterministic state.
- EAFP is strictly required across external I/O boundaries where concurrent modifications or environment races can occur.

---

## Integration and Usage

### In AI Agent Workflows
Include or reference `SKILL.md` within the agent skill configuration or context bundle to govern all Python code generation, editing, and review tasks.

### In CI / Developer Workflows
Reference the verification checklist during pull request reviews and ensure pre-commit hooks enforce formatting (`ruff format`), linting (`ruff check`), and static typing (`mypy` or `pyright`).
