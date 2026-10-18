# PR Review — JavaScript / TypeScript

**📋 USAGE: This file must be used together with `pr-review-common.instructions.md`**
## JAVASCRIPT/TYPESCRIPT TOOL DECLARATION

### Auto-Execute Tools (Run Immediately)
```yaml
static_analysis:
  - command: "tsc --noEmit"
    required: true
    failure_action: "block_review"
    output_integration: "flag_compilation_errors"
    
  - command: "npm run lint || yarn lint"
    required: true  
    failure_action: "document_issues"
    output_integration: "cross_reference_manual_findings"
    
security_scan:
  - command: "snyk test"
    required: true
    failure_action: "escalate_vulnerabilities" 
    output_integration: "flag_high_critical_cves"
    
testing:
  - command: "npm test || yarn test"
    required: true
    failure_action: "document_coverage_gaps"
    output_integration: "verify_new_code_coverage"
```

### Tool Execution Protocol
1. **Environment Check**: Verify package.json and node_modules
2. **Sequential Execution**: Run tools in dependency order
3. **Output Capture**: Store all tool outputs for analysis
4. **Integration**: Cross-reference with manual review findings
5. **Reporting**: Include tool results in final review
---

## Naming Conventions

- **Classes / Interfaces / Types / Enums**: `PascalCase`
- **Functions / methods / variables**: `camelCase`
- **Constants (module-level, never reassigned)**: `SCREAMING_SNAKE_CASE`
- **Files**: `camelCase.ts` for utilities; `PascalCase.ts` for classes/interfaces (follow existing project convention)

---

## Mandatory Tools & Analysis Workflow

### REQUIRED: Execute ALL tools before manual review

#### STEP 1: Dependency & Security Scan
```bash
# Vulnerability scan (MANDATORY)
snyk test
# Document any HIGH/CRITICAL vulnerabilities found
```

#### STEP 2: Code Quality Analysis  
```bash
# Linting (MANDATORY)
npm run lint || yarn lint
# Flag any ESLint errors not auto-fixable

# Type checking (MANDATORY)
tsc --noEmit
# Flag any TypeScript compilation errors
```

#### STEP 3: Test Coverage Analysis
```bash
# Run test suite (MANDATORY)
npm test || yarn test
# Check for test failures
# Verify coverage reports for new/modified code
```

### Tool Output Requirements
- **ALL tool failures** must be reported in review
- **Cross-reference** tool outputs with manual findings
- **Escalate** any security vulnerabilities from snyk
- **Require fixes** for TypeScript compilation errors
- **Document** test coverage gaps for new code

### Tool Prerequisites
If any tool fails to run:
1. Check package.json for script availability
2. Verify dependencies are installed
3. Document missing tools as review blocker

---

## TypeScript Best Practices

- **No `as any` casts** — require a typed alternative. Accept narrow bridges like `as Parameters<typeof Fn>[0]` with a justifying comment. Reject `as unknown as T` double-casts unless clearly explained.
- **`const` over `let`** unless reassignment is required
- **No nested ternaries** — use if/else chains for readability
- **No implicit `any`** — all function parameters and return types should be explicitly typed or unambiguously inferrable
- **No non-null assertions (`!`)** without a preceding null check or a comment explaining why null is impossible
- **Prefer `unknown` over `any`** when the type is genuinely unknown — it forces narrowing before use
- **No dead code** — flag unused imports, unreachable branches, and variables assigned but never read
- **Avoid `enum`** in favour of `const` objects with `as const` or union string types — enums have surprising JS output behaviour
- **Post-loop throw/return**: If TypeScript requires a throw/return after a loop, assess whether the loop can have zero iterations. "Unreachable at runtime" is only accurate if the input is validated — otherwise correct the comment or add validation

---

## Imports & Module Organization

- Use **path aliases** (`@utils/*`, `@workgroups/*`, etc.) — flag any relative `../` import that crosses module/workgroup boundaries
- Import order: external packages → internal aliases → relative (within same module)
- No circular imports

---

## Functions & Documentation

- **JSDoc on all exported functions** and non-trivial private functions
- `/** */` block with `@param` for every parameter (omit types — TypeScript provides them)
- `@returns` when the return value is not obvious from the function name
- Do not place documentation inside function bodies

```typescript
/**
 * Brief description of what the function does.
 *
 * @param id - The record ID to look up
 * @param cookies - Authenticated session cookies
 * @returns The parsed record, or null if not found
 */
```

---

## Error Handling

- Every `.catch()` must log with `console.error` including relevant IDs/names/paths
- `.catch()` returning `null` / `false` / `[]` — verify the caller checks for it
- Use `throw new Error(message, { cause: error })` to preserve the original stack trace
- `return` vs `continue` in loops — flag `return` where `continue` was intended; it aborts the whole loop
- Promise rejections must be handled; avoid unhandled promise rejection warnings

## Logging & Console Discipline

- **`console.error`** — failures and error conditions
- **`console.warn`** — non-critical issues (skipped items, fallbacks, degraded behaviour)
- **`console.info`** — user-visible progress messages (start/completion of major steps)
- **`console.log`** — flag any `console.log` in production code paths; it should be `console.info`, `console.warn`, or removed
- Flag any debug logging that was left in accidentally


---

## Testing (Jest)

### Test Environment & Setup
- **Verify test suites actually execute** — flag any test files that crash before running (check for runtime errors in test setup)
- **Jest setup completeness** — ensure all required polyfills are present (Web Streams, DOM globals, etc.)
- **Missing polyfills** — for Node.js environments, flag missing `ReadableStream`, `WritableStream`, `TransformStream` globals if tests import libraries that require them
- **Environment configuration** — all test environment variables must be properly configured in setup files

### Test Coverage Requirements
- **Zero coverage flagging** — explicitly flag any exported function with zero test coverage, especially:
  - HTTP-dependent functions
  - File I/O operations  
  - Error-prone parsing functions
  - Functions that return `null`/`[]` on failure
- **Coverage completeness** — all public functions must have both happy path and error path tests

### sanitizeFilename Edge Cases
All functions using `sanitizeFilename()` must test the empty string return scenario:

```typescript
it("handles sanitizeFilename returning empty string", () => {
  mockSanitize.mockReturnValue("");
  // Test that function handles empty string gracefully
  // and logs appropriate error
});
```

### Processing Loop Resilience  
Functions with processing loops must test partial failure scenarios:

```typescript
it("continues processing remaining items when one item fails", () => {
  // Mock one item to throw, others to succeed
  // Verify all successful items are processed
  // Verify error is logged for failed item
});
```

### Test scenarios
- Every happy path must be tested and assert on meaningful output values, not just "no throw"
- For parsers: use realistic HTML fixture inputs
- For dispatchers: test boundary cases — empty input, both keywords present, case-insensitivity
- For functions returning `null` / `false` / `[]` on failure: must have a test for each failure path
- For HTTP-dependent functions: must have a test where `fetch` rejects (network error)
- For retry logic: assert (a) retry happens on 5xx AND (b) **no retry on 4xx** — both must be tested

### No hacks
- **No `setTimeout` / `flushPromises` hacks** — use `jest.useFakeTimers()` + `jest.runAllTimersAsync()`
- **No `as unknown as T`** to force incompatible mock shapes — fix the mock to match the type
- **No `(global.X as jest.Mock)`** unless `jest.mock(...)` is declared at the top of the file
- Mock setup belongs in `beforeEach` / `afterEach`, not scattered inside individual tests
- **Jest environment polyfills** — flag missing Web Streams polyfills if tests import dependencies that require them:
  ```typescript
  // jest.setup.ts - Add if tests use libraries requiring Web Streams
  import { ReadableStream, WritableStream, TransformStream } from "stream/web";
  if (!globalThis.ReadableStream) globalThis.ReadableStream = ReadableStream as never;
  if (!globalThis.WritableStream) globalThis.WritableStream = WritableStream as never;
  if (!globalThis.TransformStream) globalThis.TransformStream = TransformStream as never;
  ```

---

## JavaScript/TypeScript Security Patterns

### OWASP A01 — Broken Access Control (Path Traversal)
The gold-standard pattern for any file write derived from external input:
```typescript
const fileName = sanitizeFilename(rawFileName);
if (!fileName) {
  console.error(`Sanitized filename was empty for ID ${id}: "${rawFileName}"`);
  return false;
}
const filePath = path.join(outputPath, fileName);
if (!path.resolve(filePath).startsWith(path.resolve(outputPath) + path.sep)) {
  console.error(`Path traversal detected for ID ${id}: "${rawFileName}"`);
  return false;
}
```
Flag any file write missing either `sanitizeFilename` or the `path.resolve().startsWith()` guard.

**CRITICAL**: The guard MUST include `+ path.sep` suffix. Without it, paths like `/tmp/output2/file.txt` would pass validation when `outputPath` is `/tmp/output`. This is a common security vulnerability.

### `sanitize-filename` usage
`sanitizeFilename()` can return an empty string — every call site must handle this with a fallback or skip. Flag any call site that uses the return value without checking for empty string.

### Fallback Value Validation
When sanitizeFilename() returns empty string, fallback values must also be validated:

```typescript
// WRONG - fallback bypasses sanitization
const fileName = sanitizeFilename(rawName) || `${id}.json`;

// CORRECT - validate fallback too
const fileName = sanitizeFilename(rawName) || 
                 sanitizeFilename(`${id}.json`) || 
                 "unknown.json";
```

**Flag any fallback that uses unsanitized external input** (scraped HTML, API responses, user input).

### OWASP A03 — Injection

**URL parameter injection**: Any variable (env var, extracted value) interpolated into a URL query string must use `encodeURIComponent()`.

**URL path validation**: Any `href` or path extracted from HTML and used as a URL suffix must be validated as a relative path:
```typescript
if (!href.startsWith("/") || href.startsWith("//")) {
  console.error(`Unexpected href format, skipping: ${href}`);
  continue;
}
```

**Command injection**: Never pass unsanitized input to `exec()`, `spawn()`, or `child_process` methods.

**Template injection**: Be cautious with template engines rendering user input; use auto-escaping.

### OWASP A02 — Cryptographic Failures

- No hardcoded secrets — use environment variables or secure key management
- Use `crypto.randomBytes()` for security-sensitive random values, not `Math.random()`
- Prefer modern, secure hashing algorithms (SHA-256+) over MD5/SHA-1

### OWASP A05 — Security Misconfiguration

**Environment variables**:
- Prefer `throw new Error("X is not set")` over `process.exit(1)` or silent `return` for missing required env vars
- `process.env.X || ""` followed by an emptiness check is acceptable but `throw` is preferred  
- All env vars must be documented in `.env.template` or equivalent

### Dependency Security

- Flag any `package.json` dependency pinned to a version with known CVEs
- Avoid dependencies with excessive permissions or unclear maintenance status
