# PR Review — Python

**📋 USAGE: This file must be used together with `pr-review-common.instructions.md`**

---

## Naming Conventions

- **Classes**: `PascalCase`
- **Functions / methods / variables**: `snake_case`
- **Constants (module-level)**: `SCREAMING_SNAKE_CASE`
- **Private members**: prefix with `_single_underscore`
- **"Name-mangled" members**: prefix with `__double_underscore` (use sparingly)
- **Type aliases**: `PascalCase`
- **Modules / packages**: `snake_case`

---

## Tools & Analysis

Create a virtual environment to isolate dependencies before running any tooling:
```
python3 -m venv .venv && source .venv/bin/activate
```

Install all analysis tools:
```
pip install ruff pyright pylint radon xenon vulture bandit cohesion grimp pydeps
```

### Required: Capture Tool Outputs For Inspection

To make post-review tuning easier without adding a manual review step, use a flag-style logging mode.

#### Logging mode flags
- `PY_SMELL_LOG_MODE=off` (default): run tools with normal console output and no artifact files
- `PY_SMELL_LOG_MODE=on`: persist per-tool logs and exit codes under `.review-artifacts/python-smells/`
- `PY_SMELL_LOG_LEVEL=summary|debug` (used only when mode is `on`):
    - `summary`: keep per-tool logs and generate a compact machine summary
    - `debug`: same as summary plus print each command before execution

Example:
```
PY_SMELL_LOG_MODE=on PY_SMELL_LOG_LEVEL=summary
```

#### Output artifact rules
- When enabled, store all outputs under `.review-artifacts/python-smells/`
- Capture both stdout and stderr for each tool
- Record each tool exit code in a matching `.exit` file
- Do not skip output capture when a tool fails; failed tools are still valuable signals

#### Recommended flag-driven runner
```
PY_SMELL_LOG_MODE="${PY_SMELL_LOG_MODE:-off}"
PY_SMELL_LOG_LEVEL="${PY_SMELL_LOG_LEVEL:-summary}"
ARTIFACT_DIR=".review-artifacts/python-smells"

run_smell_tool() {
    name="$1"
    shift

    if [ "$PY_SMELL_LOG_MODE" = "on" ]; then
        mkdir -p "$ARTIFACT_DIR"
        if [ "$PY_SMELL_LOG_LEVEL" = "debug" ]; then
            echo "[python-smells] running: $*"
        fi
        "$@" > "$ARTIFACT_DIR/$name.log" 2>&1
        rc=$?
        echo "$rc" > "$ARTIFACT_DIR/$name.exit"
        return "$rc"
    fi

    "$@"
}

run_smell_tool ruff ruff check . --select C90,E,F,W,N,ANN,B,SIM,RET,PLR
run_smell_tool pylint-code-level pylint . --disable=all --enable=R0801,R0912,R0913,R0914,R0915,R0902,W0611,W0612
run_smell_tool radon-cc radon cc . -s -a -n C
run_smell_tool radon-mi radon mi . -s -n B
run_smell_tool xenon xenon --max-absolute B --max-modules B --max-average A .
run_smell_tool vulture vulture . --min-confidence 80 --exclude ".venv,tests"
run_smell_tool pylint-structural pylint . --disable=all --enable=R0901,R0903,R0904,W0221,W0223,C0302
run_smell_tool cohesion cohesion -d .
run_smell_tool radon-raw radon raw . -s
run_smell_tool grimp python -m grimp your_package
run_smell_tool pydeps pydeps your_package --show-cycles --noshow --max-bacon=3

if [ "$PY_SMELL_LOG_MODE" = "on" ] && [ "$PY_SMELL_LOG_LEVEL" = "summary" -o "$PY_SMELL_LOG_LEVEL" = "debug" ]; then
    {
        echo "Python Smell Tool Output Summary"
        for f in "$ARTIFACT_DIR"/*.exit; do
            [ -e "$f" ] || continue
            tool="$(basename "$f" .exit)"
            code="$(cat "$f")"
            status="PASS"
            [ "$code" -eq 0 ] || status="FAIL"
            echo "$tool: $status (exit=$code)"
        done
    } > "$ARTIFACT_DIR/summary.txt"
fi
```

#### Agent behavior requirement
- Do not require a manual "human review" summary section for smell tools
- If logging mode is `on`, reference `.review-artifacts/python-smells/summary.txt` and per-tool logs
- If logging mode is `off`, proceed normally without artifact expectations

---

### Code-Level Smell Detection

These tools target the most granular smells: long methods, too many branches, duplicate code, dead code, and feature envy proxies.

**ruff** — fast, actively maintained linter covering style, correctness, and many pylint smell rules:
```
ruff check . --select C90,E,F,W,N,ANN,B,SIM,RET,PLR
```
Key rule groups: `C90` = McCabe complexity · `B` = bugbear (common Python anti-patterns) · `SIM` = unnecessary complexity · `RET` = return statement smells · `PLR` = pylint refactor rules (too many args, branches, statements)

**pylint** — deeper design smell checks not yet covered by ruff; target the `R` (refactor) and `W` (warning) codes specifically:
```
pylint . --disable=all --enable=R0801,R0912,R0913,R0914,R0915,R0902,W0611,W0612
```
`R0801` = duplicate code · `R0912` = too many branches · `R0913` = too many arguments · `R0914` = too many local variables · `R0915` = too many statements · `R0902` = too many instance attributes

**radon** — cyclomatic complexity and maintainability index; use as a diagnostic, not a gate:
```
radon cc . -s -a -n C
radon mi . -s -n B
```
`-n C` shows only functions ranked C or below · `-n B` shows modules below a B maintainability grade

**xenon** — hard complexity budget gate suitable for CI; fails the build if thresholds are crossed:
```
xenon --max-absolute B --max-modules B --max-average A .
```

**vulture** — dead code; flag unused functions, classes, and variables:
```
vulture . --min-confidence 80 --exclude ".venv,tests"
```

---

### Structural Smell Detection

These tools target class/module-level design issues: high coupling, low cohesion, deep inheritance, oversized classes.

**pylint** — structural design warnings:
```
pylint . --disable=all --enable=R0901,R0903,R0904,W0221,W0223,C0302
```
`R0901` = too many ancestors (deep inheritance) · `R0903` = too few public methods (anemic class) · `R0904` = too many public methods (God class proxy) · `W0221` = signature mismatch on override · `C0302` = too many lines in module

**cohesion** — LCOM (Lack of Cohesion of Methods); low scores indicate a class doing too many unrelated things:
```
cohesion -d .
```

**radon** — class-level raw metrics to identify structural bloat (LOC, number of methods, number of attributes):
```
radon raw . -s
```

---

### Architectural Smell Detection

These tools identify system-wide design issues: cyclic dependencies, scattered functionality, unstable dependency direction, and God objects at the package level.

**grimp** — builds an import graph to detect dependency cycles and layering violations:
```
python -m grimp your_package
```
Raises `CyclicImportError` on cycles · Use the Python API to check dependency layers when a layered architecture (e.g. `api` → `services` → `domain`) is enforced

**pydeps** — visualizes package dependency structure; review cycle output without opening the browser:
```
pydeps your_package --show-cycles --noshow --max-bacon=3
```
`--show-cycles` highlights only cyclic paths · `--max-bacon=3` limits graph depth to avoid noise

---

### Security & Dependency Checks

**bandit** — static security analysis; `-r` is recursive, `-ll` reports medium and high severity only, `--exclude` skips the venv and tests:
```
bandit -r . -ll --exclude ".venv,tests"
```

**snyk** — dependency vulnerability scanning against CVE databases:
```
snyk test
```

---

### Type Checking

**pyright** — strict, actively maintained, fast; preferred for new codebases:
```
pyright .
```

---

### PR Quality Gate Expectations

- A PR should fail if it introduces **new** findings at ruff `E`/`F`/`PLR`, pylint `R`/`W`, xenon threshold breach, bandit medium+ severity, or grimp cycle.
- A PR that intentionally accepts a finding must include a short justification in the PR description (not a `# noqa` comment alone).
- Prospector can be used as an aggregator alternative to running pylint and pycodestyle separately; use a committed `.prospector.yaml` profile to keep suppressions in version control.

## Python Best Practices

- **Type hints required** on all function signatures (PEP 484) — parameters and return types
- **No bare `except:`** — always catch a specific exception type; bare `except` catches `SystemExit`, `KeyboardInterrupt`, and `GeneratorExit`
- **No mutable default arguments** — `def f(x=[])` is a classic bug; use `None` sentinel and assign inside the function
- **No `type: ignore` comments** without an explanation of why the suppression is necessary
- **`is` vs `==`** — use `is` only for identity comparisons (`None`, `True`, `False`); use `==` for value equality
- **`f-strings` over `.format()` and `%`** for string formatting (Python 3.6+)
- **Context managers** (`with`) for all file I/O and resource acquisition — never leave resources open on exception paths
- **No `global` or `nonlocal`** unless absolutely necessary and clearly commented
- **Comprehensions over `map`/`filter`** for readability, but avoid nested comprehensions beyond two levels
- **`pathlib.Path` over `os.path`** for new file path manipulation code
- **No feature envy patterns** — when a function repeatedly manipulates another object/module's data, move logic closer to that data owner
- **No shotgun surgery patterns** — avoid changes that require touching many scattered files for one behavior change; centralize behavior behind clear module boundaries
- **Keep branch complexity bounded** — split condition-heavy functions into smaller strategies/helpers when branch count grows
- **Avoid unstable dependency direction** — higher-level policy modules must not depend on lower-level implementation details in reverse
- **Prevent scattered functionality** — keep related business rules in cohesive modules instead of spreading them across unrelated packages
- **Prefer cohesive classes/modules** — if methods share little state or purpose, split into smaller components
- **Avoid redundant abstractions** — remove wrappers/layers that add no behavior, validation, or boundary value

---

## Imports & Module Organization

- Follow **PEP 8 import order**: stdlib → third-party → local, each group separated by a blank line
- No wildcard imports (`from module import *`) — they pollute the namespace and obscure origins
- No unused imports — flag any import that is never referenced

---

## Functions & Documentation

- **Docstrings on all public functions, classes, and modules** (PEP 257)
- Use **Google-style docstrings**:

```python
def fetch_data(record_id: str, cookies: list[str]) -> dict | None:
    """Fetches a record from the API by ID.

    Args:
        record_id: The unique identifier of the record.
        cookies: Authenticated session cookies.

    Returns:
        The parsed record dict, or None if not found.

    Raises:
        HTTPError: If the request fails with a non-retryable status.
    """
```

- Private / internal helpers do not require full docstrings but should have a one-liner if non-obvious

---

## Error Handling

- Use **exception chaining** (`raise NewError("msg") from original_error`) to preserve the original traceback — do not swallow or replace exceptions silently
- Log with `logging.error(msg, exc_info=True)` to capture the full traceback — avoid bare `print()` for error conditions
- Do not catch an exception just to re-raise it unchanged — let it propagate naturally
- Validate inputs early (`if not x: raise ValueError(...)`) rather than letting AttributeError / KeyError surface later
- Functions returning `None`/empty values on failure must be checked by callers

## Logging & Console Discipline

- Use the `logging` module — not `print()` — in all library/module code
- **`logging.error`** — failures and error conditions
- **`logging.warning`** — non-critical issues, skipped items, fallbacks
- **`logging.info`** — user-visible progress (start/completion of major steps)
- **`logging.debug`** — verbose diagnostic output, always gated; never on by default in production
- Flag any `print()` in production code paths that should be `logging.*`

---

## Testing (pytest)

### Test Environment & Setup
- **Verify test suites actually execute** — flag any test files that crash before running (check for import errors, missing dependencies)
- **Test environment completeness** — ensure all required test dependencies and fixtures are available
- **Environment configuration** — all test environment variables must be properly configured

### Test Coverage Requirements
- **Zero coverage flagging** — explicitly flag any exported function with zero test coverage, especially:
  - HTTP-dependent functions
  - File I/O operations
    - Error-prone parsing functions
  - Functions that return `None`/empty values on failure
- **Coverage completeness** — all public functions must have both happy path and error path tests

### Test scenarios
- Every happy path must assert on the output value, not just that no exception was raised
- For parsers: use realistic fixture inputs (HTML, JSON, CSV files in `tests/fixtures/`)
- For functions returning `None` / empty list / `False` on failure: must have tests for each failure path
- For HTTP-dependent functions: must mock the HTTP layer and test both success and network-error paths
- For retry logic: assert (a) retry happens on 5xx AND (b) **no retry on 4xx**

### No hacks
- **No `time.sleep()` in tests** — use `freezegun`, `pytest-mock` timers, or mock `time.sleep` directly
- **No `unittest.mock.patch` string targets that are wrong** — patch where the name is *used*, not where it is *defined*
- **No broad `mock.patch("module.*")`** — patch at the narrowest scope possible
- Fixtures belong in `conftest.py`; do not duplicate fixture setup across test files
- Use `pytest.raises(ExceptionType, match=r"pattern")` to assert both the exception type and the message

---

## Python Security Patterns

### OWASP A03 — Injection

**SQL injection**: Always use parameterised queries — never string-interpolate values into SQL:
```python
# BAD
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
# GOOD
cursor.execute("SELECT id, name, email FROM users WHERE id = %s", (user_id,))
```

**Command injection**: Never pass unsanitised strings to `subprocess`, `os.system`, `eval`, or `exec`. Use `subprocess.run([...], shell=False)` with an explicit argument list.

**Template injection**: Flag any use of `eval()`, `exec()`, or Jinja2/similar templates rendering unsanitised user input without sandboxing.

### OWASP A01 — Broken Access Control (Path Traversal)

Any filename or path derived from external input must be validated before use:
```python
import pathlib

safe_base = pathlib.Path(output_dir).resolve()
candidate = (safe_base / untrusted_filename).resolve()
if not str(candidate).startswith(str(safe_base) + "/"):
    raise ValueError(f"Path traversal detected: {untrusted_filename}")
```

Flag any `open()` or `Path.write_*` call where the path includes external input without this guard.

### OWASP A02 — Cryptographic Failures

- **No `hashlib.md5` or `hashlib.sha1`** for security purposes (integrity checks of trusted data is acceptable)
- **No hardcoded secrets** — credentials, API keys, tokens must come from environment variables or a secrets manager
- **No `random` module for security** — use `secrets` for tokens, nonces, or anything security-sensitive

### OWASP A05 — Security Misconfiguration

- All environment variables must be documented in `.env.template` or `README`
- Prefer raising an exception over silent fallback when a required env var is missing:
```python
api_key = os.environ.get("API_KEY")
if not api_key:
    raise EnvironmentError("API_KEY environment variable is not set")
```

### Dependency Security

- Flag any `requirements.txt` or `pyproject.toml` dependency pinned to a version with known CVEs
- Flag unpinned (`package` with no version specifier) production dependencies
