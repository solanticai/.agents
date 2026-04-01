---
name: project-audit
description: Comprehensive project completion audit and code quality review. Use this skill whenever the user asks to review, audit, verify, or check their project — especially phrases like "review my project", "audit completed work", "check all tasks are done", "final review", "pre-launch check", "verify everything is complete", "QA my code", "check for bugs", "is my project finished", "run a project audit", or any request to validate that a codebase meets its original plan, requirements, or todo list. Also trigger when users mention "hardening", "cleanup", "deprecated code", "security review", "performance check", or "build check". This skill performs a multi-phase audit covering task completion, code quality, bugs, types, security, performance, deprecated code cleanup, and build verification.
---

# Project Audit Skill

You are performing a rigorous, multi-phase audit of a software project. The goal is to verify that all planned work is complete, all code is correct and hardened, and the project is production-ready.

## Before You Start

1. **Locate the project plan/requirements.** Look for files like `TODO.md`, `PLAN.md`, `README.md`, `CHANGELOG.md`, `requirements.md`, `.cursor/rules`, `CLAUDE.md`, or any planning documents in the project root and common locations. Also check for GitHub issues references, Jira-style task IDs, or inline `TODO`/`FIXME`/`HACK` comments in source code. Ask the user to point you to their plan if you can't find one.

2. **Identify the tech stack.** Scan `package.json`, `requirements.txt`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `composer.json`, `Gemfile`, or equivalent to understand the language, framework, and tooling. This determines which build, lint, and type-check commands to run.

3. **Establish the project structure.** Run a directory tree (excluding `node_modules`, `.git`, `dist`, `build`, `__pycache__`, `.next`, `venv`, `.venv`) to understand the codebase layout.

## Audit Phases

Execute each phase in order. For each phase, report findings using the format in the "Reporting" section below. Do not skip phases — if a phase is not applicable (e.g., no build step), note it as N/A and move on.

---

### Phase 1: Task & Requirements Completion Audit

**Objective:** Verify every planned task, feature, and requirement has been implemented to 100%.

Steps:
1. Read the project plan, todo list, or requirements document in full.
2. Create a checklist of every discrete task, feature, or requirement mentioned.
3. For each item, search the codebase to confirm it has been implemented. Look at the actual code — don't rely on commit messages or comments alone.
4. Flag any item that is partially complete, missing, or implemented differently than specified.
5. Check for orphaned TODO/FIXME/HACK/XXX comments that indicate unfinished work:
   ```bash
   grep -rn "TODO\|FIXME\|HACK\|XXX\|@todo\|PLACEHOLDER\|TEMP\|STUB" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" --include="*.py" --include="*.rs" --include="*.go" --include="*.php" --include="*.rb" --include="*.java" --include="*.cs" --include="*.vue" --include="*.svelte" . | grep -v node_modules | grep -v .git | grep -v dist | grep -v build
   ```
6. Summarise completion status: total tasks, completed, incomplete, and partially complete.

---

### Phase 2: Type Safety & Static Analysis

**Objective:** Confirm all types are correct and there are no type errors anywhere in the codebase.

Steps:
1. Run the appropriate type checker for the project:
   - **TypeScript:** `npx tsc --noEmit` (or the project's configured tsconfig)
   - **Python:** `mypy .` or `pyright .` (check pyproject.toml for config)
   - **Rust:** `cargo check`
   - **Go:** `go vet ./...`
   - Other languages: use the standard type-checking tool
2. Run the project's configured linter if one exists:
   - **JS/TS:** `npx eslint .` or check package.json scripts for a lint command
   - **Python:** `ruff check .` or `flake8 .`
   - **Rust:** `cargo clippy`
   - **Go:** `golangci-lint run`
3. Report every error. Do not dismiss warnings without explanation.
4. If the project has no type system, note this and move on.

---

### Phase 3: Bug & Logic Audit

**Objective:** Identify bugs, logic errors, and incorrect behaviour through manual code review.

Steps:
1. Review every source file that contains application logic (skip config files, lock files, and generated code).
2. For each file, check for:
   - **Off-by-one errors** in loops, array access, pagination, and slicing
   - **Null/undefined handling** — unguarded access to values that could be null
   - **Race conditions** in async code — missing awaits, unsynchronised shared state
   - **Error handling gaps** — catch blocks that swallow errors silently, missing try/catch around I/O
   - **Incorrect conditional logic** — flipped boolean conditions, missing edge cases, unreachable branches
   - **Resource leaks** — unclosed file handles, database connections, event listeners not cleaned up
   - **Incorrect API usage** — wrong HTTP methods, missing headers, incorrect payload shapes
   - **State management issues** — stale closures, incorrect dependency arrays (React), mutation of state that should be immutable
3. If test files exist, run the test suite and report results:
   ```bash
   # Find and run the test command from package.json, Makefile, or equivalent
   ```
4. Check for dead code — functions, components, or modules that are defined but never imported or called.

---

### Phase 4: Code Structure & Optimisation

**Objective:** Ensure code is well-structured, maintainable, and performs efficiently.

Steps:
1. **Architecture review:**
   - Is there a clear separation of concerns (e.g., routes/controllers, services, data access)?
   - Are there any god files (>500 lines) that should be split?
   - Is shared logic properly abstracted — no significant copy-paste duplication?
2. **Import hygiene:**
   - Unused imports across all source files
   - Circular dependency risks
   ```bash
   # For JS/TS projects:
   npx madge --circular --extensions ts,tsx,js,jsx src/ 2>/dev/null || echo "madge not available — skip circular check"
   ```
3. **Performance review:**
   - N+1 query patterns in database access
   - Missing indexes for frequent queries (if schema files exist)
   - Unnecessary re-renders in React components (missing memo/useMemo/useCallback where it matters)
   - Unoptimised loops or data transformations on large datasets
   - Missing pagination or unbounded data fetches
   - Large bundle imports where tree-shaking could help (e.g., importing all of lodash)
4. **Recommended improvements:** List specific, actionable improvements with file paths and line numbers.

---

### Phase 5: Failsafes & Guardrails

**Objective:** Verify that appropriate defensive programming patterns are in place.

Steps:
1. **Input validation:** Are all user inputs validated and sanitised before processing? Check API endpoints, form handlers, and CLI argument parsers.
2. **Error boundaries:** Are there proper error boundaries in the UI (React ErrorBoundary, Vue errorCaptured, etc.)?
3. **Rate limiting:** Are API endpoints protected against abuse (rate limiting, throttling)?
4. **Timeouts:** Do external API calls, database queries, and network requests have timeouts configured?
5. **Graceful degradation:** Does the application handle failures in external dependencies gracefully (fallbacks, retries with backoff, circuit breakers)?
6. **Data integrity:** Are database writes wrapped in transactions where atomicity matters? Are there constraints at the database level (not just application level)?
7. **Logging & observability:** Is there sufficient logging for debugging production issues without exposing sensitive data in logs?
8. **Environment guards:** Are environment variables validated at startup? Are required secrets checked for presence before the app runs?

---

### Phase 6: Security Audit

**Objective:** Identify security vulnerabilities and flaws.

Steps:
1. **Secrets & credentials:**
   ```bash
   # Check for hardcoded secrets, API keys, passwords
   grep -rn "password\s*=\|api_key\s*=\|secret\s*=\|token\s*=\|PRIVATE_KEY" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.py" --include="*.env" --include="*.yaml" --include="*.yml" --include="*.json" . | grep -v node_modules | grep -v .git | grep -v dist | grep -v "*.lock"
   ```
   Check `.gitignore` ensures `.env`, credentials, and key files are excluded.
2. **Dependency vulnerabilities:**
   ```bash
   # JS/TS
   npm audit 2>/dev/null || yarn audit 2>/dev/null
   # Python
   pip-audit 2>/dev/null || safety check 2>/dev/null
   ```
3. **Injection vulnerabilities:** Check for SQL injection (raw queries with string interpolation), XSS (unsanitised user content rendered as HTML), command injection, path traversal.
4. **Authentication & authorisation:** Verify auth checks on protected routes/endpoints. Ensure role-based access control is enforced server-side, not just client-side.
5. **CORS & headers:** Check CORS configuration is restrictive (not `*` in production). Verify security headers (CSP, X-Frame-Options, etc.) are set.
6. **Data exposure:** Ensure API responses don't leak sensitive fields (passwords, internal IDs, PII that shouldn't be exposed). Check that error messages don't expose stack traces or internal details in production mode.

---

### Phase 7: Feature Hardening

**Objective:** Verify that implemented features are robust and handle edge cases.

Steps:
1. For each major feature identified in Phase 1:
   - **Empty states:** Does the UI handle zero-data scenarios gracefully?
   - **Loading states:** Are loading indicators shown during async operations?
   - **Error states:** Are user-facing errors clear and actionable?
   - **Boundary conditions:** Maximum input lengths, file size limits, pagination limits
   - **Concurrent access:** Can multiple users interact with the same resource without conflicts?
   - **Idempotency:** Are operations safe to retry (especially payment flows, form submissions)?
2. Check that all user-facing strings are consistent and professional (no placeholder text like "Lorem ipsum" or "TODO: write copy").

---

### Phase 8: Deprecated Code Cleanup

**Objective:** Ensure no deprecated files, dead code, or legacy artefacts remain.

Steps:
1. **Orphaned files:** Identify files that are not imported or referenced anywhere:
   ```bash
   # For JS/TS — find source files not imported by anything
   # Review the output manually — config files and entry points will show as "unreferenced" but are valid
   ```
2. **Commented-out code:** Find large blocks of commented-out code (>5 lines) that should be removed:
   ```bash
   grep -rn "^[[:space:]]*//" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" . | grep -v node_modules | grep -v .git | head -50
   ```
3. **Deprecated API usage:** Check for usage of deprecated library methods or framework patterns (e.g., React class components when the project uses hooks, deprecated Next.js APIs).
4. **Unused dependencies:** Compare `package.json` (or equivalent) dependencies against actual imports:
   ```bash
   # JS/TS
   npx depcheck 2>/dev/null || echo "depcheck not available"
   ```
5. **Build artefacts committed to repo:** Check for `dist/`, `build/`, `.next/`, `__pycache__/`, `*.pyc` in version control.
6. **Stale configuration:** Unused config entries, environment variables referenced in config but never used in code.

---

### Phase 9: Build & Type Verification

**Objective:** Confirm the project builds cleanly with zero errors and zero warnings.

Steps:
1. Run the full build:
   ```bash
   # Detect and run the project's build command
   # Check package.json "scripts.build", Makefile, Cargo.toml, etc.
   ```
2. Capture and report the full output. A successful audit requires:
   - **Zero errors**
   - **Zero warnings** (or explicitly justified warnings)
3. If the project has multiple build targets (e.g., client and server, or multiple packages in a monorepo), build each one.
4. If the project has a production build mode, use it (e.g., `npm run build` not `npm run dev`).
5. Run the type check again after the build to confirm no regressions:
   ```bash
   npx tsc --noEmit 2>&1 || echo "Type check completed with errors"
   ```

---

## Reporting

After completing all phases, produce a structured audit report using this format:

```
# Project Audit Report

## Summary
- **Project:** [name]
- **Date:** [date]
- **Overall Status:** PASS / FAIL / PASS WITH WARNINGS
- **Tasks Complete:** X/Y (Z%)

## Phase Results

### Phase 1: Task Completion — [PASS/FAIL]
[Checklist of all tasks with status]

### Phase 2: Type Safety — [PASS/FAIL]
[Type check and lint results]

### Phase 3: Bug Audit — [PASS/FAIL/WARNINGS]
[List of issues found with severity, file, and line]

### Phase 4: Code Structure — [PASS/WARNINGS]
[Structural issues and improvement recommendations]

### Phase 5: Failsafes — [PASS/FAIL]
[Missing guardrails and recommendations]

### Phase 6: Security — [PASS/FAIL/CRITICAL]
[Vulnerabilities found with severity rating]

### Phase 7: Feature Hardening — [PASS/WARNINGS]
[Unhardened features and edge cases]

### Phase 8: Deprecated Cleanup — [PASS/FAIL]
[Orphaned files, dead code, stale dependencies]

### Phase 9: Build Verification — [PASS/FAIL]
[Build output and type check results]

## Critical Issues (must fix)
[Numbered list — security vulnerabilities, bugs, broken features]

## Recommended Improvements (should fix)
[Numbered list — performance, structure, hardening]

## Minor Suggestions (nice to have)
[Numbered list — code style, optimisations, cleanup]
```

## Important Principles

- **Be thorough, not superficial.** Read the actual code. Run the actual commands. Don't scan file names and guess.
- **Be specific.** Every finding must include a file path and line number (or range). Vague findings like "consider improving error handling" are not useful — say exactly where and what.
- **Don't fix things during the audit.** The audit is a report, not a refactor. List what needs fixing and let the user decide how to proceed.
- **Distinguish severity clearly.** Critical (must fix before deploy), Warning (should fix soon), and Suggestion (improvement opportunity) are different things.
- **Verify, don't assume.** If the plan says "implement auth middleware" and you see an auth file, actually read it to confirm it works correctly — don't just check the box because the file exists.
- **Run commands when possible.** If you can run `tsc`, `npm run build`, `npm audit`, `grep`, etc., do it. Real command output is more reliable than eyeballing code.
