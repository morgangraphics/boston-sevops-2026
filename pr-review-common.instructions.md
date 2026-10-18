# PR Review — Common Process & Security (Language-Agnostic)

## TOOL FRAMEWORK DECLARATION

### Required Tool Categories
```yaml
auto_execute:
  - gh_cli_comments    # Always run for PR context
  - github_mcp_server  # Always run for file analysis

conditional_execute:
  - language_specific  # Run based on file extensions detected
  - security_scanners  # Run for security review phase

validation_required:
  - ALL tool outputs must be processed before review completion
  - Tool failures must be documented as review blockers
  - Cross-reference tool findings with manual inspection
```

### Tool Integration Protocol
1. **Pre-Review**: Execute auto_execute tools
2. **Language Detection**: Execute applicable language-specific tools
3. **Analysis Phase**: Process all tool outputs
4. **Manual Review**: Cross-validate findings
5. **Report Generation**: Integrate tool and manual findings

---

## IMPORTANT: Complete Review Requirements

**MANDATORY FILE USAGE:**
- **ALWAYS** use this common file together with the appropriate language-specific file:
  - For Python: Use `pr-review-common.instructions.md` + `pr-review-python.instructions.md`
  - For JavaScript/TypeScript: Use `pr-review-common.instructions.md` + `pr-review-js-ts.instructions.md`
  - For Terraform: Use `pr-review-common.instructions.md` + `pr-review-terraform.instructions.md`

**MANDATORY THREE-PHASE REVIEW:**
1. **Phase 1 - Primary Code Review** (Claude Sonnet and ChatGPT Codex):
  - Follow both common + language-specific instructions
  - Focus: code quality, best practices, testing, documentation
2. **Phase 2 - Cross-Model Verification & Vetting** (Claude Sonnet and ChatGPT Codex):
  - ChatGPT Codex: pass all Phase 1 findings to Claude Sonnet for verification
  - Claude Sonnet: pass all Phase 1 findings to ChatGPT Codex for verification
  - Resolve disagreements explicitly; keep only findings confirmed by at least one model after challenge/rebuttal
  - Produce one consolidated, vetted findings set to carry into Phase 3
  - **Phase gate**: do not start Phase 3 until both model-to-model verification passes are completed
3. **Phase 3 - Security Review** (Claude Opus):
  - Focus: OWASP Top 10, STRIDE threat modeling, vulnerability assessment
  - Read Phase 2 results first to avoid duplication

**ALL PHASES MUST BE COMPLETED** - Do not skip the security review.

---

## Review Workflow & Model Strategy

### Multi-Model Approach

**For Python PRs:**
1. **Primary Review**: Claude Sonnet and ChatGPT 5.3 Codex
2. **Cross-Model Verification**: Claude Sonnet and ChatGPT 5.3 Codex
3. **Security Review**: Claude Opus

**For JavaScript/TypeScript PRs:**
1. **Primary Review**: Claude Sonnet and ChatGPT 5.3 Codex
2. **Cross-Model Verification**: Claude Sonnet and ChatGPT 5.3 Codex
3. **Security Review**: Claude Opus

**For Terraform/Terragrunt PRs:**
1. **Primary Review**: Claude Sonnet and ChatGPT 5.3 Codex
2. **Cross-Model Verification**: Claude Sonnet and ChatGPT 5.3 Codex
3. **Security Review**: Claude Opus

### Review Focus Areas
- **Primary Review**: Code quality, best practices, testing, syntax conventions, anti-patterns, documentation
- **Cross-Model Verification**: Mutual vetting of Phase 1 findings, conflict resolution, and consolidation of validated issues
- **Security Review**: OWASP Top 10 analysis, STRIDE threat modeling, vulnerability assessment

### Coordination
- Primary reviewers must exchange findings and complete cross-model verification before security review starts
- Security reviewer should read the cross-model vetted findings first
- Focus security analysis on areas not covered by primary review
- Cross-validate any conflicting recommendations

## Review Process

- Read **all** existing review thread comments on the PR, including resolved threads, before making any recommendations. Do not re-flag issues that have already been acknowledged and fixed.
- Validate author explanations for technical accuracy — do not accept justifications at face value if they are incorrect or misleading.
- Where a previous reviewer's comment was wrong, say so clearly and explain why.
- Consider the project context when calibrating severity — an internal CLI tool has a different threat surface than a public-facing web service.

---

## Mandatory Tool Usage & Workflow

### STEP 1: PR Context Analysis (Required)
**BEFORE** reviewing any code, execute these tools:

1. **Pull Request Comments Review**:
  ```bash
  gh pr view {PR_NUMBER} --json comments,reviews
  ```
  - Parse ALL existing comments (including resolved threads)
  - Identify issues already flagged and their status
  - Note author responses and explanations
   - **DO NOT re-flag resolved issues**

2. **Repository Data Access**:
  - Use GitHub MCP Server tools to:
    - Get PR file changes and diff context
    - Access project structure and dependencies
    - Retrieve related commit history if needed

### STEP 2: Language-Specific Analysis (Required)
**MUST** execute all applicable tools defined in language-specific instruction files:
- See `pr-review-js-ts.instructions.md` for JavaScript/TypeScript mandatory tools
- See `pr-review-python.instructions.md` for Python mandatory tools
- See `pr-review-terraform.instructions.md` for Terraform/Terragrunt mandatory tools

### STEP 3: README Provisioning / Setup Verification (Required)
Validate that `README.md` setup instructions are runnable and accurate for a clean development environment.

#### Execution rules
- Execute setup/provisioning/configuration steps from `README.md` in order
- If a step fails, log the failure and continue to the next step
- Do not stop at first error; capture complete setup fidelity
- Persist command output and failures under `.review-artifacts/readme-provisioning/`

#### Isolation rules (to protect local environment)
- **Python**: use a project-local virtual environment (`python3 -m venv .venv`), activate it before installs/tests, and avoid global `pip` installs
- **JavaScript/TypeScript**: use project-local dependencies (`npm ci` / `pnpm install --frozen-lockfile` / `yarn install --frozen-lockfile`); do not install packages globally
- **Terraform/Terragrunt**: run in workspace-local context (`.terraform/`), avoid changing global CLI config, and prefer version managers (`tfenv`/`tgenv`) when version pinning is required
- If README omits environment isolation guidance, log it as a documentation gap

#### Recommended logging pattern
```bash
mkdir -p .review-artifacts/readme-provisioning

run_readme_step() {
  step_name="$1"
  shift
  "$@" > ".review-artifacts/readme-provisioning/${step_name}.log" 2>&1
  rc=$?
  echo "$rc" > ".review-artifacts/readme-provisioning/${step_name}.exit"
  return 0  # Continue even on failure
}
```

### Tool Output Integration
- **ALL tool outputs** must be analyzed before writing review comments
- **Cross-reference** tool findings with manual code review
- **Validate** that tool warnings align with instruction file requirements
- **Document** any tool failures or limitations in review notes
- **Add README provisioning findings** to categorized output under `Setup/Provisioning Verification`


## Output Format

For every finding, produce:

PR #{number} Review — Consolidated Report
Reviewers: Claude Sonnet · ChatGPT {version} Codex · Claude Opus (Security)

1. **Severity**: 🔴 Critical / 🟡 Medium / 🔵 Low
2. **Status**: `✅ Confirmed Fixed` / `🔁 Partially Fixed` / `⚠️ Still Outstanding` / `🆕 New`
3. **Title**: short description
4. **Copy+Paste GitHub URL** with line highlights to the PR file where the issue occurs, e.g.:
  `https://github.com/{org}/{repo}/pull/{pr_number}/changes#diff-{hash}#L10-L35`
5. **Technical assessment**: explain the issue precisely and why it matters
6. **Copy-paste fix**: ready-to-apply code block

Do not pad with noise. Only flag genuine issues. Group findings by severity.

---

## Data Preservation & API Contract Validation

### Interface/API Changes
When reviewing code that modifies interfaces, data structures, or replaces existing functionality:

- **Validate data preservation**: Ensure all relevant fields from the original implementation are preserved or explicitly documented as intentionally removed
- **Check for silent data loss**: Flag any refactoring where data extraction logic is removed without replacement
- **Interface completeness**: When new interfaces replace old ones, verify all previously extracted/stored data is accounted for
- **Migration documentation**: Changes that drop data fields must be explicitly documented and justified
- **Comment preservation**: Pay attention to comments like "Some data needs to be split/extracted" — ensure equivalent functionality exists in replacement code

**Pattern to flag**: Code deletions that include data extraction logic or field mapping without equivalent functionality in replacement code.

---

## Universal Error Handling Principles

- Never silently swallow errors — always log with appropriate level, including relevant context (IDs, names, file paths)
- Functions that return `null`/empty values on failure must be handled by callers — verify the caller checks for these conditions
- Preserve original error context when re-throwing or wrapping exceptions
- `return` vs `continue` in loops — ensure loop control flow matches intent (early `return` aborts entire loop)
- Validate inputs early rather than allowing deeper AttributeError/TypeError exceptions

### Loop Error Containment
- **Per-item error handling**: Processing loops (forms, attachments, records) must use per-item try/catch to prevent one failure from aborting the entire batch
- **Graceful degradation**: Loop processing should continue with remaining items when individual items fail
- **Error logging**: Each failed item should be logged with identifying information (ID, name, index)

**Required pattern for processing loops**:
```typescript
for (const item of items) {
  try {
    await processItem(item);
  } catch (error) {
    console.error(`Failed to process item ${item.id}:`, error);
    // Continue with remaining items
  }
}
```

---

## Security — OWASP Top 10 & STRIDE

### OWASP A01 — Broken Access Control (Path Traversal)
Any filename or path derived from external input (HTML, API response, user input) must be:
1. Sanitized with a library or allowlist before use
2. Validated to ensure it stays within expected directory boundaries before writing

Flag any file write that skips either step. See language-specific files for implementation examples.

### OWASP A03 — Injection

**URL parameter injection**: Any variable interpolated directly into a URL query string must be percent-encoded. Flag raw interpolation without encoding.

**Command injection**: Never pass un-sanitized external input to shell commands (`exec`, `spawn`, `subprocess`, etc.).

**Cookie / header injection**: Any string interpolated into a cookie or HTTP header value must not contain control characters (`;`, `\r\n`, `=`). If all callers are compile-time constants, note it is defensible but flag for future-proofing.

### OWASP A05 — Security Misconfiguration

**Environment variables**: Every env var used in the codebase must be documented (e.g., in `.env.template` or `README`). Flag undocumented variables. Prefer explicit error handling over silent failures or abrupt process termination when required env vars are missing.

### OWASP A09 — Security Logging Failures

- Debug/verbose logging must be gated behind an environment flag that defaults to off
- Gated logs must not expose credential values (tokens, passwords, secrets)

### STRIDE

| Threat | Watch for |
|---|---|
| **Spoofing** | Unauthenticated requests, session fixation |
| **Tampering** | Unvalidated file paths, unencoded URL params, unsanitized filenames |
| **Repudiation** | Missing error logging, swallowed exceptions |
| **Information Disclosure** | Credential logging, stack traces in user-facing output |
| **Denial of Service** | Unbounded loops/recursion, no depth/retry limits |
| **Elevation of Privilege** | Path traversal writing outside expected directory |
