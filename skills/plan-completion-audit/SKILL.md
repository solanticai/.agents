---
name: plan-completion-audit
description: Full-stack audit after all plan phases are complete. Triggers on "audit completed work", "plan is complete", "full-stack audit", "audit supabase", or "check frontend and backend align". Verifies plan vs code, frontend bugs, Supabase schema, RLS, and type alignment.
argument-hint: [path-to-project-root-or-plan-file]
allowed-tools: Read, Grep, Glob, Write, Edit, Bash, WebSearch, WebFetch, Agent
user-invocable: true
---

# Plan Completion Audit

You are performing a rigorous, full-stack audit after all phases of a project plan have been completed. The goal is to verify that every planned task was implemented correctly, the frontend is bug-free and hardened, and the Supabase backend is correctly structured and aligned with the frontend application.

## Before You Start

1. **Locate the plan.** Find the project plan, task list, or requirements document. Check for: `PLAN.md`, `TODO.md`, `TASKS.md`, `README.md`, `CHANGELOG.md`, `requirements.md`, `.cursor/rules`, `CLAUDE.md`, GitHub issues, or inline `TODO`/`FIXME`/`HACK` comments. If no plan is found, ask the user to provide one — do not proceed without a plan to audit against.

2. **Identify the tech stack.** Scan `package.json`, `pyproject.toml`, `tsconfig.json`, or equivalent. Determine the framework (Next.js, React, Python/FastAPI, etc.), database (Supabase, Postgres), and tooling (ESLint, Prettier, tsc, etc.).

3. **Map the project structure.** Run:
   ```bash
   find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" -o -name "*.py" -o -name "*.sql" \) | grep -v node_modules | grep -v .next | grep -v dist | grep -v .git | sort
   ```

4. **Determine Supabase access method.** Try in this order:
   - **Supabase CLI** (preferred): Check if `supabase` CLI is available and a project is linked (`supabase status`)
   - **Supabase MCP**: Check if a Supabase MCP server/connector is available in the current environment
   - **Supabase Management API**: Use as last resort if CLI and MCP are unavailable
   
   If none are available, note it and perform the backend audit using local migration files and type definitions only.

## Audit Phases

Execute every phase in order. Report findings per phase using the format in the Reporting section. Never skip a phase — mark as N/A if genuinely not applicable.

---

### Phase 1: Plan Completion Verification

**Objective:** Confirm every task, feature, and requirement from the plan has been implemented to 100%.

1. Read the full plan/requirements document.
2. Build a checklist of every discrete task, feature, or deliverable.
3. For each item, search the codebase and verify the implementation exists and is functional — read the actual code, do not just confirm a file exists.
4. Flag items that are: missing, partially complete, or implemented differently than specified.
5. Scan for unfinished work markers:
   ```bash
   bash "${CLAUDE_PLUGIN_ROOT}/scripts/check-todos.sh" .
   ```
   Or if the script is unavailable, run the grep manually:
   ```bash
   grep -rn "TODO\|FIXME\|HACK\|XXX\|PLACEHOLDER\|TEMP\|STUB\|@todo\|INCOMPLETE\|WIP" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" --include="*.py" --include="*.sql" . | grep -v node_modules | grep -v .git | grep -v dist | grep -v .next
   ```
6. Summarise: total tasks, completed, incomplete, partially complete.

---

### Phase 2: Type Safety & Static Analysis

**Objective:** Zero type errors and zero lint violations across the entire codebase.

Run the automated check if available:
```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/check-types.sh" .
```

Or manually:
1. **TypeScript:** `npx tsc --noEmit 2>&1`
2. **ESLint:** `npx eslint . 2>&1` or the project's configured lint command from `package.json`
3. **Python (if applicable):** `mypy .` or `pyright .`
4. Report every error. Do not dismiss warnings without justification.
5. Check for `any` type abuse — flag files with excessive `as any`, `// @ts-ignore`, or `// @ts-expect-error` that suppress real issues.

---

### Phase 3: Bug & Logic Audit

**Objective:** Identify bugs, logic errors, and broken functionality through manual code review.

Review every source file containing application logic. For each file check:
- **Null/undefined:** Unguarded access to values that could be null, especially data from Supabase queries
- **Async errors:** Missing `await`, unhandled promise rejections, race conditions
- **Error handling:** Empty catch blocks, swallowed errors, missing try/catch around I/O and database calls
- **Conditional logic:** Flipped booleans, missing edge cases, unreachable branches
- **State management:** Stale closures in React, incorrect useEffect dependency arrays, mutation of state
- **API misuse:** Wrong HTTP methods, incorrect payload shapes, missing auth headers
- **Off-by-one errors:** Pagination, array slicing, loop boundaries
- **Resource leaks:** Unclosed subscriptions, listeners not cleaned up in useEffect returns

If tests exist, run them:
```bash
npm test 2>&1 || npx jest 2>&1 || npx vitest run 2>&1 || echo "No test runner found"
```

Check for dead code — functions, components, hooks, or utilities that are defined but never imported.

---

### Phase 4: Code Structure & Optimisation

**Objective:** Confirm code is well-structured, maintainable, and performant.

1. **Architecture:**
   - Clear separation of concerns (routes, components, services, hooks, utils, types)?
   - God files over 500 lines that should be split?
   - Copy-paste duplication that should be abstracted?

2. **Import hygiene:**
   - Unused imports across all files
   - Circular dependencies:
     ```bash
     npx madge --circular --extensions ts,tsx,js,jsx src/ 2>/dev/null || echo "madge not available"
     ```

3. **Performance:**
   - N+1 query patterns in Supabase calls (multiple sequential queries that should be joins or single RPCs)
   - Missing `useMemo`/`useCallback` where expensive computations re-run on every render
   - Unbounded data fetches without pagination or limits
   - Large bundle imports (full lodash, full date-fns, etc.)
   - Unnecessary re-renders — components subscribing to state they don't use

4. List specific improvements with file paths and line numbers.

---

### Phase 5: Failsafes & Guardrails

**Objective:** Verify defensive programming patterns are in place.

1. **Input validation:** All user inputs validated before processing — forms, API params, URL params, file uploads
2. **Error boundaries:** React ErrorBoundary components wrapping major UI sections
3. **Loading states:** Every async operation has a loading indicator
4. **Empty states:** UI handles zero-data scenarios gracefully (no blank screens)
5. **Error states:** User-facing errors are clear, actionable, and don't expose internals
6. **Rate limiting:** API routes protected against abuse where applicable
7. **Timeouts:** External API calls and long-running operations have timeouts
8. **Graceful degradation:** Failures in non-critical services don't crash the app
9. **Environment guards:** Required env vars validated at startup, not at first use
10. **Data integrity:** Supabase writes use transactions where atomicity matters

---

### Phase 6: Security Audit

**Objective:** Identify security vulnerabilities.

Run the automated check:
```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/check-secrets.sh" .
```

Then manually verify:
1. **Secrets:** No hardcoded API keys, passwords, tokens, or private keys in source
2. **`.gitignore`:** `.env`, `.env.local`, credential files, and key files are excluded
3. **Dependency vulnerabilities:** `npm audit 2>&1`
4. **Injection:** No raw SQL with string interpolation, no unsanitised HTML rendering (dangerouslySetInnerHTML without sanitisation), no command injection
5. **Auth & authorisation:** Protected routes require auth, RBAC enforced server-side via Supabase RLS — not just client-side checks
6. **CORS:** Not set to `*` in production config
7. **Data exposure:** API responses and Supabase queries don't leak sensitive fields, error messages don't expose stack traces in production
8. **Supabase-specific:** RLS enabled on all user-facing tables, service role key never exposed to client, anon key permissions are minimal

---

### Phase 7: Feature Hardening

**Objective:** Verify features are robust and handle edge cases.

For each major feature from Phase 1:
1. **Empty states:** Zero-data UI is handled
2. **Loading states:** Spinners/skeletons during async operations
3. **Error states:** Clear, actionable error messages
4. **Boundary conditions:** Max input lengths, file size limits, pagination limits enforced
5. **Concurrent access:** Multiple users editing the same resource won't corrupt data
6. **Idempotency:** Duplicate form submissions, payment retries, and webhook replays are safe
7. **Placeholder text:** No "Lorem ipsum", "TODO: write copy", or "test" strings in production UI
8. **Accessibility basics:** Interactive elements are keyboard-accessible, images have alt text, form fields have labels

---

### Phase 8: Deprecated Code Cleanup

**Objective:** No orphaned files, dead code, or legacy artefacts remain.

Run the automated check:
```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/check-deprecated.sh" .
```

Then verify:
1. **Orphaned files:** Source files not imported or referenced anywhere
2. **Commented-out code:** Blocks of >5 lines of commented code that should be removed
3. **Deprecated APIs:** Usage of deprecated framework/library methods (e.g., deprecated Next.js APIs, old React patterns)
4. **Unused dependencies:**
   ```bash
   npx depcheck 2>/dev/null || echo "depcheck not available"
   ```
5. **Build artefacts in repo:** `dist/`, `build/`, `.next/`, `__pycache__/` committed to source
6. **Stale config:** Unused env vars, orphaned config entries
7. **Old migration files:** Supabase migrations that were superseded but not cleaned up (check `supabase/migrations/`)

---

### Phase 9: Build Verification

**Objective:** The project builds cleanly with zero errors and zero warnings.

1. Run the full production build:
   ```bash
   npm run build 2>&1 || yarn build 2>&1 || pnpm build 2>&1
   ```
2. Capture full output. A pass requires **zero errors** and **zero unacknowledged warnings**.
3. If the project has multiple build targets (e.g., monorepo packages), build each one.
4. Run a final type check after the build:
   ```bash
   npx tsc --noEmit 2>&1
   ```
5. If a dev server or preview mode exists, verify it starts without errors:
   ```bash
   timeout 15 npm run dev 2>&1 || echo "Dev server check skipped"
   ```

---

### Phase 10: Supabase Backend Audit

**Objective:** Verify all database tables, RPC functions, RLS policies, triggers, and indexes are correct, complete, and aligned with the application.

This phase is critical. Read `${CLAUDE_PLUGIN_ROOT}/references/supabase-audit-guide.md` for the full checklist before starting.

**10a. Schema Inspection**

Retrieve the current database schema. Use the first available method:

```bash
# Option 1: Supabase CLI
bash "${CLAUDE_PLUGIN_ROOT}/scripts/audit-supabase.sh"

# Option 2: If CLI unavailable, inspect local migration files
ls -la supabase/migrations/ 2>/dev/null
cat supabase/migrations/*.sql 2>/dev/null
```

For each table, verify:
- Column names, types, and nullability match what the frontend expects
- Primary keys and foreign keys are correctly defined
- Default values are appropriate
- Indexes exist for columns used in WHERE clauses and JOIN conditions
- `created_at` / `updated_at` timestamps exist where expected

**10b. RPC Function Audit**

List all RPC functions and for each one:
- Verify the function signature (params and return type) matches how the frontend calls it
- Check the function body for SQL correctness
- Confirm `SECURITY DEFINER` vs `SECURITY INVOKER` is set appropriately
- Verify `search_path` is set to prevent search path injection
- Check that functions called from the client are exposed via the API schema (public/exposed)

**10c. RLS Policy Audit**

For every table that stores user data:
- RLS must be enabled (`ALTER TABLE ... ENABLE ROW LEVEL SECURITY`)
- Policies must exist for SELECT, INSERT, UPDATE, DELETE as appropriate
- Policies must reference `auth.uid()` or equivalent — no policies that return `true` for all users on sensitive tables
- Service role bypasses are intentional and documented

**10d. Triggers & Realtime**

- Verify any database triggers are correctly defined and fire on the expected events
- If the app uses Supabase Realtime, confirm the relevant tables have replication enabled
- Check that realtime subscriptions in the frontend match the tables configured for replication

**10e. Storage & Edge Functions**

If applicable:
- Storage buckets have appropriate RLS policies
- File size limits and allowed MIME types are configured
- Edge functions (if any) are deployed and their endpoints match frontend calls

---

### Phase 11: Frontend ↔ Backend Alignment

**Objective:** Confirm the frontend application and Supabase backend are in complete agreement — types, queries, RPC calls, and data flow.

1. **Type alignment:** Compare Supabase-generated types (e.g., `database.types.ts` or output from `supabase gen types typescript`) against the types used in frontend code. Flag mismatches in:
   - Table row types vs frontend interfaces
   - RPC function param/return types vs frontend call sites
   - Enum values in the database vs enum values in the frontend

2. **Query audit:** For every Supabase client call in the frontend (`.from()`, `.rpc()`, `.select()`, `.insert()`, `.update()`, `.delete()`):
   - The table or function exists in the database
   - The columns referenced in `.select()` exist on the table
   - The filter columns in `.eq()`, `.in()`, `.match()` etc. exist and are the correct type
   - Insert/update payloads include all required (non-nullable, no-default) columns
   - The query result is typed correctly in the consuming code

3. **Auth flow alignment:** The frontend auth implementation (sign up, sign in, sign out, session refresh, password reset) aligns with the Supabase auth configuration. Check that:
   - Auth providers configured in Supabase match what the frontend offers
   - Protected routes check session state correctly
   - Token refresh is handled (not relying on expired tokens)

4. **Missing backend objects:** Search the frontend for any `.from('table_name')` or `.rpc('function_name')` calls that reference tables or functions that don't exist in the database.

5. **Missing frontend consumers:** Check for database tables or RPC functions that were created as part of the plan but are never called from the frontend (possibly indicating incomplete integration).

---

## Reporting

After all phases, produce a structured report. Use the template in `templates/audit-report.md` as the base structure.

The report must include:
- A clear PASS / FAIL / PASS WITH WARNINGS verdict per phase
- Every finding must include a **file path and line number** (or table/function name for backend findings)
- Severity ratings: **CRITICAL** (must fix), **WARNING** (should fix), **SUGGESTION** (nice to have)
- A prioritised action list at the end

## Important Principles

- **Be thorough.** Read actual code and run actual commands. Don't scan file names and guess.
- **Be specific.** Every finding needs a file path + line number or a table/function name. "Consider improving error handling" is useless — say exactly where and what.
- **Don't fix during the audit.** This is a report, not a refactor. List findings and let the user decide.
- **Verify, don't assume.** If the plan says "implement org-level access control" and you see an auth file, read it to confirm it works — don't check the box because the file exists.
- **Run commands.** Use the scripts in `scripts/` and run build/lint/type commands. Real output beats eyeballing.
- **Cross-reference constantly.** The frontend and backend must agree. A table without a consumer or a query against a non-existent column are both failures.
