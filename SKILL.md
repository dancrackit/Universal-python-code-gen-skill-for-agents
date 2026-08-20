---
name: universal-python-code-gen
description: Comprehensive, production-ready Python coding standard and code generation rules for AI agents. Enforces strict precedence (Inspect project -> Correctness -> Security -> Robustness -> Performance -> Style), concurrency-safe error handling (LBYL vs. EAFP), modern static typing, zero import-time I/O, and architectural guardrails across greenfield and existing Python projects.
---

# Python Coding Standard for AI Agents

Precedence when rules conflict: **Inspect project → Correctness → Security → Robustness → Performance → Style.**

This order is deliberate: a fast, stylish function that's wrong or unsafe is worse than a plain one that isn't.

*v2 incorporates a cross-check against Dagster's public "Dignified Python" agent rules — see the reconciliation note near the bottom for what changed and why.*

---

## 1. Inspect the project before writing anything

Check, in order: Python version (`pyproject.toml` / `.python-version`), existing formatter/linter/type checker, existing test framework, existing module layout and import conventions, and dependency manager.

Follow existing conventions unless they are clearly unsafe (e.g. bare `except:`, hardcoded secrets, shell-injectable subprocess calls). Do not silently swap a project's `pandas` for `polars` or `json` for `orjson` just because it's "more modern" — that's an unrequested rewrite, not a fix, and it will get bounced in review on someone else's repo.

For genuinely new/greenfield projects with no existing convention: `uv` for packages, `Ruff` for lint/format, `mypy` or `pyright` for types, `pytest` for tests.

## 2. Correctness first, performance second

Write the simple, readable version that solves the actual requirement. Optimize only when input size or latency demands it, the code is already performance-sensitive, or profiling shows a real bottleneck.

`orjson` and `polars` are good choices *when the data volume justifies them* — not defaults for every script.

## 3. Magic methods and properties must be O(1)

`__len__`, `__bool__`, `__contains__`, and any `@property` are called implicitly and often repeatedly — in loops, conditionals, membership checks. They must never hide I/O, a network call, or an iteration over a large collection.

```python
# WRONG - __len__ doing O(n) work on every call
def __len__(self) -> int:
    return sum(1 for _ in self._items)

# CORRECT - O(1)
def __len__(self) -> int:
    return self._count
```

If a value genuinely requires computation or I/O to produce, make it an explicit method (`get_row_count()`), not a property or magic method - the explicit method signals "this might be slow" in a way `len(x)` never will.

## 4. Error handling - the part most guides get wrong

**Rule:** Use LBYL only for checks where nothing external can change between the check and the use - validating a function argument's type, checking a key in a dict the code itself owns and controls. Use exceptions (EAFP) for anything touching state that can change out from under you: files, sockets, databases, subprocesses, other processes, external APIs.

**Why this split, not "LBYL always":** `if path.exists(): path.read_text()` is a time-of-check-to-time-of-use (TOCTOU) race. Between the check and the read, the file can be deleted by another process, a cleanup job, or a container restart - and now you get an *unhandled* `FileNotFoundError` because the code was written as if the check made it safe. This is a real, recurring production failure class, not a style preference.

```python
# WRONG - TOCTOU race, will break under concurrency
if path.exists():
    data = path.read_text(encoding="utf-8")

# CORRECT - the failure path is explicit and handled
try:
    data = path.read_text(encoding="utf-8")
except FileNotFoundError:
    logger.warning("Expected file missing: %s", path)
    raise
```

For state you own and control - a local dict, a config object you just built - LBYL is fine and often clearer:

```python
# FINE - no external actor can invalidate this between the check and the use
if key in mapping:
    process(mapping[key])
```

Other error-handling rules:
- Never use a bare `except:`. Never `except Exception: pass` - log or handle meaningfully.
- Catch specific exceptions only (`requests.exceptions.ConnectionError`, not `Exception`).
- Let unexpected exceptions bubble to the CLI/API/job boundary; don't wrap everything internally.
- When re-raising inside `except`, use `raise ... from e` (or `from None` if deliberately suppressing the chain) so the original cause isn't lost.

## 5. Typing

- Fully type-hint all public function signatures (params and return).
- Modern syntax: `list[str]`, `dict[str, int]`, `str | None` - not `List`/`Optional`.
- Avoid `Any` unless bridging an untyped boundary (e.g. an untyped C extension or third-party stub gap).
- Use `Literal` or enums for fixed string sets instead of bare strings - catches typos at type-check time.
- `typing.cast()` is compile-time only - it tells the type checker to trust you but verifies nothing at runtime. When the check is cheap (an `isinstance` call), assert before casting instead of casting blind:

  ```python
  # WRONG - blind cast, silent misbehavior if the assumption is wrong
  cast(dict[str, Any], doc)["key"] = value

  # CORRECT - the assumption is actually verified
  assert isinstance(doc, MutableMapping), f"Expected MutableMapping, got {type(doc)}"
  cast(dict[str, Any], doc)["key"] = value
  ```

  Skip the assertion only right after a type guard has already proved it, or in a measured, documented hot path.

## 6. Structural constraints (guardrails for agent-generated code)

- Max 4 levels of indentation per function - extract a helper if you exceed it.
- Return early to reduce nesting.
- Keep functions single-purpose, but don't fragment code into needless one-line wrappers.
- Declare variables close to first use, not pre-declared at the top of the function.
- Parameters: keep the first one or two positional (`self`, `ctx`, the primary subject); force everything else keyword-only with a `*` separator once a function has 5 or more params. This scales better than a hard numeric cap - a function can legitimately need six well-named keyword args more than it needs a dataclass wrapper for three. If the keyword-only list itself grows unwieldy (roughly 7+) or the args form one cohesive unit of config, bundle them into a `dataclass` instead.
- Avoid default parameter values unless the default is correct for the large majority of callers. A default is a silent behavior choice - the caller who needed something else will forget to override it and get a bug that doesn't announce itself. (This is why §8 requires `encoding` to be passed explicitly at every call site rather than relying on `read_text()`'s platform-dependent default.) Acceptable defaults: truly optional behavior correct 95%+ of the time, or a temporary default kept for backward compatibility on an existing public API.

## 7. Imports

- Top of module, absolute imports only, no wildcard imports, no re-exports (`__all__` gymnastics) - import from the canonical location.
- **No import-time I/O.** No network calls, env reads, config loads, or client construction at module scope. Import-time I/O breaks testability, causes circular import chains, and slows cold starts - all things that surface as "the app is flaky in prod" bugs that are hard to trace back to an import line. Defer with a function or a cached factory:

  ```python
  from functools import cache
  from pathlib import Path

  @cache
  def _session_file_path() -> Path:
      """Path to the session ID file, computed once, on first call."""
      return Path("scratch/current-session-id")
  ```

## 8. Paths and files

- `pathlib.Path` exclusively - never `os.path`.
- Always specify `encoding="utf-8"` explicitly on read/write.
- Two different failure modes, two different rules:
  - **Before `.resolve()` or `.is_relative_to()`**: check `.exists()` first. These are path-string operations, not content reads - nothing external is racing you between the check and the call, so there's no meaningful window for the answer to go stale, and checking first avoids surprising behavior on missing paths or broken symlinks.
  - **Before reading or writing file *content*** (`.read_text()`, `open()`, etc.): don't treat `.exists()` as a safety gate. Between the check and the actual read, another process can delete or replace the file - a real race in any environment with concurrent workers, cleanup jobs, or container restarts. Let the read's own exception handling be the safety net instead (see §4).

## 9. Security

- Never hardcode secrets, tokens, or credentials - use environment variables or a secret manager.
- Never log or print URLs, headers, or payloads that contain secrets, auth tokens, or PII.
- Validate untrusted input at every boundary (API, CLI, file upload, webhook payload).
- Subprocesses: argument lists only (never a shell string built from user input), `check=True`, an explicit `timeout`, and captured/logged output.
- Avoid `eval`/`exec`, unsafe deserialization (`pickle` on untrusted data), path traversal, and overly broad file permissions.
- Use `with` statements for every external resource (files, sockets, locks, DB connections) so cleanup is guaranteed even on exception.

## 10. Dependencies

- Add a dependency only when it materially reduces risk or complexity - prefer the standard library for small tasks.
- Record dependencies in `pyproject.toml`.
- For OSS contributions specifically: match the target repo's existing dependency choices. A PR that adds `orjson` to swap out `json` for a 200-line script is friction the maintainer didn't ask for.

## 11. Testing

- Add or update tests for meaningful behavior, edge cases, and regressions - not implementation trivia.
- Mock external boundaries: APIs, databases, the clock, randomness, the filesystem.
- Use temp directories/files for test artifacts.
- Run the smallest relevant tests first, then broader suites when the change's risk warrants it.

## 12. Documentation

Google-style docstrings on all public functions and classes. Keep comments updated when logic changes - a stale comment is worse than no comment.

## 13. Verification before handoff

Before calling anything done, run (or state clearly why you couldn't):
- Formatter/linter (Ruff, or the project's existing tool)
- Type checker (mypy/pyright, or the project's existing tool)
- Tests (pytest, or the project's existing runner)
- A security-focused pass: secrets, injection points, unsafe logging, unsanitized I/O

If verification can't be run, say exactly what wasn't run and why - don't imply it passed.

---

## Reconciled against Dagster's "Dignified Python"

Dagster's public LLM-agent rules (dagster.io, Jan 2026) also lead with LBYL-by-default, which is worth checking against §4 rather than dismissing outright. On inspection, their own code examples are narrower than a blanket rule and don't actually conflict with it:

- Their LBYL example is a dict-membership check on **state the code itself owns**, with no external actor that can mutate it between the check and the use - exactly the case §4 already calls safe for LBYL.
- Their `exists()`-before-`.resolve()` example doesn't touch file *content* - it's the narrower, legitimate carve-out described in §8, not the TOCTOU-prone pattern below.
- They explicitly carve out exceptions at error boundaries and third-party APIs that force try/except - functionally the same scope §4 gives to EAFP.

So "LBYL by default," read as "LBYL for state you own," and this guide's split aren't in conflict. What's still rejected is the *blanket* version - check-then-read a file's content with no exception handling at all - because that specific pattern breaks under concurrency regardless of which source recommends it.
