# Verify Command (Hybrid Phase 5)

## Path Resolution (IMPORTANT)

This command operates on the CURRENT PROJECT, not global .claude/.

When reading/writing files:

- `.claude/implementations/issue-123.md` = `{project}/.claude/implementations/issue-123.md`
- NOT `~/.claude/implementations/issue-123.md`

## GitHub Repository Resolution (CRITICAL)

**Before any GitHub operations, determine the correct repository:**

1. **Check CLAUDE.md** (in project root):
   - Look for line starting with `**Repository:**`
   - This is the PREFERRED method - project explicitly declares its repo

2. **Fall back to git remote**:
   - Run: `git remote get-url origin`
   - Parse: `git@github.com:owner/repo.git` -> `owner/repo`
   - Or: `https://github.com/owner/repo.git` -> `owner/repo`

3. **Store as {github_repo}** for all GitHub operations

**NEVER hardcode a repository name. Always resolve dynamically.**

## Purpose

Hybrid Phase 5: Testing, Fixes & Commit. Addresses GPT's review findings, runs full test suite, and commits.

**What this command does:**

- Reads GPT's review and addresses issues
- Adds any additional tests GPT recommended
- Runs targeted security pre-check (authorization surface, secrets, rate limits)
- Verifies any new database migrations are applied to the dev DB (catches schema-cache mismatches at runtime)
- Runs full test suite, linting, and type checking
- Verifies acceptance criteria
- Creates git commit

**What this command does NOT do:**

- Push to remote (user does this manually)
- Re-run GPT's review
- Make architectural changes (go back to /architect if needed)

---

## Execution Instructions

When the user says "/verify #N" or "Verify issue #N", execute these steps:

### Step 1: Parse Issue Number

```
EXTRACT: Issue number from user's message
FORMAT: "#N", "issue N", "verify N"
STORE: {issue_number}
```

### Step 2: Read and Validate Issue File

```
ACTION: Read .claude/implementations/issue-{issue_number}.md

VALIDATE:
1. File exists
   IF NOT: OUTPUT "No issue file found for #{issue_number}." STOP

2. Mode is "Hybrid"
   IF NOT: OUTPUT "Issue #{issue_number} is not in Hybrid mode. Use '/commit' for single-model." STOP

3. "## GPT Review" section exists
   IF NOT: OUTPUT "No GPT review found. GPT needs to run '/review #{issue_number}' first." STOP

4. "## Implementation Notes" section exists
   IF NOT: OUTPUT "No implementation found. Run '/build #{issue_number}' first." STOP

EXTRACT:
- GPT Review: Critical issues, Important issues, Minor issues
- GPT Review: Additional tests recommended
- GPT Review: Recommendation (APPROVE / REQUEST CHANGES / BLOCK)
- Original requirements (for acceptance criteria)
- Implementation notes (files changed, tests written)
```

### Step 3: Assess GPT's Recommendation

```
IF recommendation == "BLOCK":
  OUTPUT: "GPT's review BLOCKED this implementation.

  Blocking reasons:
  {list critical issues}

  Options:
  1. Address the blocking issues now (recommended)
  2. Go back to GPT for revised architecture: '/architect #{issue_number}'
  3. Override block and proceed anyway (not recommended)

  Reply: fix / architect / override"

  WAIT FOR USER RESPONSE:
    IF "fix": PROCEED to Step 4
    IF "architect": STOP with message to re-run architecture
    IF "override": PROCEED to Step 4 with warning logged

IF recommendation == "REQUEST CHANGES":
  OUTPUT: "GPT requested changes. Addressing issues now..."
  PROCEED to Step 4

IF recommendation == "APPROVE":
  OUTPUT: "GPT approved the implementation. Running final validation..."
  PROCEED to Step 5 (skip Step 4)
```

### Step 4: Address Review Issues

```
ACTION: Fix issues from GPT's review

4a. CRITICAL issues (MANDATORY):
  FOR EACH critical issue:
    READ: The file and line referenced
    UNDERSTAND: The problem and suggested fix
    IMPLEMENT: The fix
    TRACK: What was fixed

  IF a critical fix causes test failures:
    ATTEMPT: Fix the test (up to 3 tries)
    IF still failing:
      OUTPUT: "Fixing GPT's critical issue at {file}:{line} caused test failures.

      The fix: {description}
      Test failure: {error}

      Options:
      1. Iterate on the fix
      2. Skip this fix (document as known issue)
      3. Consult GPT again

      Reply: iterate / skip / consult"

      WAIT FOR USER RESPONSE

4b. IMPORTANT issues:
  FOR EACH important issue:
    ASSESS: Is this straightforward to fix?
    IF yes: Fix it
    IF no or unclear:
      OUTPUT: "GPT flagged an important issue:
      {issue description}

      This fix is non-trivial. Should I:
      1. Fix it now
      2. Skip it (document as known limitation)

      Reply: fix / skip"

      WAIT FOR USER RESPONSE

4c. MINOR issues:
  ASSESS: Are any quick to fix (< 5 lines)?
  IF yes: Fix the quick ones
  Skip the rest (document as deferred)

4d. Additional tests GPT recommended:
  FOR EACH recommended test:
    WRITE: The test as specified
    RUN: The individual test to verify it passes
```

### Step 4.5: Security Pre-Check

```
ACTION: Run targeted security checks before testing and committing.
This catches the most common real-world vulnerabilities that GPT review may miss.
The checks are focused on patterns from known breaches in AI-built apps.

NOTE: This is NOT a full Security Agent review. For deep security analysis
(STRIDE, OWASP Top 10, attack scenarios), use /security-phase.
This step catches the highest-impact, most-common vulnerabilities.

NOTE: These checks are stack-agnostic. Project-specific overlays
(e.g. row-level-security audits, payment-webhook signature checks, framework-
specific public env-var prefix lists) belong in your project's CLAUDE.md or
AGENTS.md so the core command stays portable.

4.5a. Authorization Surface Check:
  SEARCH: For new/modified database tables, access policies, or auth middleware
  CHECK: Are sensitive fields (subscription_status, plan, tier, rate_limit,
         credits, tokens, role, is_admin, is_premium) stored on records that
         the end user can write directly?
  CHECK: Do all tables/collections with user data have access policies enforced
         at the data layer (not just at the application layer)?
  CHECK: Are admin/service-tier credentials used only in server-side code paths?

  IF sensitive fields are user-writable:
    FLAG: "🔴 CRITICAL: {field} on user-writable record — users can modify
    their own subscription/rate limits via direct API access.
    Move to a restricted record with read-only access from the user."

4.5b. Client-Side Secret Scan:
  SEARCH: For environment variable usage in frontend/client code
  CHECK: Are any variables with public/client-exposed prefixes referencing
         secret material? Common public prefixes include `NEXT_PUBLIC_`,
         `VITE_`, `EXPO_PUBLIC_`, `REACT_APP_`, `PUBLIC_`. Project-specific
         prefixes belong in the project CLAUDE.md.
  CHECK: Are AI provider keys (OpenAI, Anthropic, Replicate, etc.) in client code?
  CHECK: Are payment-provider secret keys (e.g. `sk_` style keys, client secrets)
         in client code?
  CHECK: Is `.env` in `.gitignore`?

  IF violations found:
    FLAG: "🔴 CRITICAL: {key type} exposed in client-side code"

4.5c. Rate Limiting Check (especially for AI features):
  SEARCH: For AI/LLM endpoint handlers (API routes calling AI providers)
  CHECK: Do these endpoints have server-side rate limiting?
  CHECK: Are rate limits enforced per-user AND per-IP?
  CHECK: Are rate limit counters stored where users CANNOT modify them?

  IF AI endpoints exist without backend rate limits:
    FLAG: "🟡 IMPORTANT: AI endpoint {path} has no server-side rate limiting.
    Frontend rate limits are bypassed via direct API calls."

4.5d. Sensitive API Call Check:
  SEARCH: For direct AI/payment/email service calls in frontend code
  CHECK: Is any frontend file importing or calling:
    - AI SDKs (openai, anthropic, replicate, etc.)
    - Payment SDKs with secret keys
    - Email services (sendgrid, postmark, resend) with API keys
    - Cloud storage with credentials (aws-sdk with hardcoded keys)

  IF found:
    FLAG: "🔴 CRITICAL: Sensitive API call from client code — must go
    through a server-side handler (API route, edge function, or backend service)."

4.5e. Financial Controls Check:
  CHECK: Are budget caps or billing alerts mentioned in deployment config?
  NOTE: This is informational — document whether financial safeguards exist.

RESULTS:
  IF any 🔴 CRITICAL flags:
    OUTPUT: "⚠️ Security Pre-Check found critical issues:
    {list flags}

    These are commonly exploited vulnerabilities in production apps.

    Options:
    1. Fix now before proceeding (recommended)
    2. Proceed to testing anyway (acknowledge risk)
    3. Run full security review: /security-phase #{issue_number}

    Reply: fix / proceed / full-review"

    WAIT FOR USER RESPONSE:
      IF "fix": Address the flagged issues, then re-run 4.5
      IF "proceed": Log warning, PROCEED to Step 5
      IF "full-review": STOP with instruction to run /security-phase

  IF only 🟡 IMPORTANT flags:
    OUTPUT: "Security Pre-Check: {count} concerns noted (not blocking).
    {list flags}
    Proceeding to test suite."
    PROCEED to Step 5

  IF no flags:
    OUTPUT: "Security Pre-Check: ✅ No common vulnerability patterns found."
    PROCEED to Step 4.6
```

### Step 4.6: Migration Application Check (database-backed projects)

```
ACTION: Check that any database migrations introduced by this issue are actually
applied to the dev database, not just present as files in the repo. This catches
the "code references a column that doesn't exist yet" failure mode where the
migration file passed code review but never ran against the DB (causing schema-
cache mismatches at runtime — e.g. PostgREST PGRST204, Prisma client errors,
ORM "table not found" errors).

GATE: This check only runs if the project has a migration directory AND the
issue introduced or modified files in it. The migration directory is declared
in the project CLAUDE.md (commonly: `supabase/migrations/`, `prisma/migrations/`,
`db/migrate/`, `migrations/`, `alembic/versions/`, etc.). If the project does
not declare one, skip this step.

DETECT LOCAL MIGRATION FILES FOR THIS ISSUE:
  PARSE: Implementation Notes section of issue file
  EXTRACT: Files under {migrations_dir} listed as Create / Modify / Rename
  ALSO CHECK: `git status --porcelain {migrations_dir}` for any added/modified
              migration files not yet committed
  STORE: {issue_migration_files} as list of file paths

IF {issue_migration_files} is empty:
  OUTPUT: "Migration Check: N/A (no migration files in this issue)."
  PROCEED to Step 5

TOOL DISCOVERY:
  The project's CLAUDE.md should declare how migrations are applied
  (CLI command, MCP tool, or dashboard). Check it for:
    - Apply command (e.g. `supabase db push`, `prisma migrate deploy`,
      `alembic upgrade head`, `rake db:migrate`)
    - List-applied command/MCP for verifying applied state
    - Any project-specific safety conventions

  IF no automated apply path is documented:
    OUTPUT: "⚠️ Migration Check: No automated migration application path
    documented in CLAUDE.md. Cannot verify migration application state
    automatically.

    Migration files in this issue:
    {list issue_migration_files}

    Reminder: ensure these migrations are applied to your dev database before
    merging this PR. Apply via your project's documented workflow."
    PROCEED to Step 5

FETCH APPLIED MIGRATIONS:
  EXECUTE the project's documented list-applied command (or MCP tool)
  STORE: {applied_versions} as set of version identifiers from results

COMPARE:
  FOR EACH file in {issue_migration_files}:
    EXTRACT: version identifier from filename (timestamp prefix, semantic
             version, or whatever convention the project uses)
    EXTRACT: migration name from filename
    IF version identifier NOT IN {applied_versions}:
      ADD to {missing_migrations} as {file, version, name}

IF {missing_migrations} is empty:
  OUTPUT: "Migration Check: ✅ All {count} migration(s) for this issue are
  applied to the dev DB."
  PROCEED to Step 5

IF {missing_migrations} is not empty:
  OUTPUT: "⚠️ Migration Check: {count} migration file(s) in this issue have NOT
  been applied to the dev database yet:

  {for each missing migration: list filename, version, name}

  Without these migrations applied, the runtime code in this issue will fail
  with schema-cache or 'column not found' errors as soon as it reads or writes
  the new schema.

  Options:
  1. Apply now via the project's documented apply command (recommended for
     additive, non-destructive migrations)
  2. Show migration contents first, then decide
  3. Skip — I will apply manually before merging (acknowledged risk)

  Reply: apply / show / skip"

  WAIT FOR USER RESPONSE:

    IF "show":
      FOR EACH missing migration: Read the migration file and display its contents
      Re-prompt with apply / skip

    IF "apply":
      FOR EACH missing migration in version-order (ascending):
        READ: the migration file contents

        SAFETY SCAN: check for potentially destructive patterns appropriate to
        the migration language:
          - DROP TABLE / DROP COLUMN (without "IF EXISTS")
          - TRUNCATE
          - Unconditional DELETE / data-loss UPDATE
          - Type changes that may lose data
          - DROP NOT NULL on a populated column
          - DROP of a unique/primary key constraint

        IF any destructive pattern found:
          OUTPUT: "🔴 Migration {name} contains a potentially destructive
          operation:
          {snippet of the matched pattern}

          I will NOT auto-apply this. Review and apply manually via the
          project's CLI or dashboard after confirming the data impact."
          SKIP this migration, continue to next

        ELSE:
          EXECUTE the project's documented apply command for this migration
          IF success:
            LOG: "Applied {version}_{name}"
          IF failure:
            OUTPUT: "Migration {name} failed: {error}

            Options:
            1. Abort verify (recommended — fix the migration first)
            2. Continue with remaining migrations (risky)

            Reply: abort / continue"
            WAIT FOR USER RESPONSE

      AFTER applying all eligible migrations:
        RE-FETCH applied versions and verify each succeeded
        OUTPUT: "Migration Check: ✅ Applied {success_count} migration(s).
        Schema cache will refresh per the framework's normal lifecycle.
        {if any were skipped due to safety scan: list them as manual TODOs}"
        PROCEED to Step 5

    IF "skip":
      LOG: "User chose to skip migration application."
      RECORD: Add to the verification section that will be appended in Step 7:
        "### Known Deploy Gap
        The following migration(s) are present in the repo but were NOT applied
        to the dev database during /verify. They MUST be applied before merge:
        {list of unapplied migration files with version + name}"
      OUTPUT: "Migration Check: ⚠️ Skipped — gap recorded in verification
      section. Apply manually before merge."
      PROCEED to Step 5
```

### Step 5: Run Full Test Suite

```
ACTION: Run comprehensive validation. The exact commands are declared in the
project's CLAUDE.md (Development Workflow → Common Tasks). Use whichever of
the following sub-steps apply to the languages/tooling the project uses.

5a. Backend tests (if backend code changed):
  EXECUTE: the project's documented backend test command
           (e.g. `pytest`, `go test ./...`, `bundle exec rspec`)
  STORE: {backend_test_result} (pass/fail, count)

5b. Frontend tests (if frontend code changed):
  EXECUTE: the project's documented frontend test command
           (e.g. `npm test`, `pnpm test`, `yarn test`, `vitest run`)
  STORE: {frontend_test_result} (pass/fail, count)

5c. Type check (if a typed language with a separate type-check step is in use):
  EXECUTE: the project's documented type-check command
           (e.g. `npx tsc --noEmit`, `mypy .`, `flow check`)
  STORE: {typecheck_result} (pass/fail)

5d. Linting (if a linter is configured):
  EXECUTE: the project's documented lint command
           (e.g. `npm run lint`, `ruff check`, `golangci-lint run`)
  STORE: {lint_result} (pass/warnings/errors)

5e. Frontend verification (browser-automation MCP) - if frontend files changed:

  GATE: Check if any changed files are under the project's frontend directory
  (declared in CLAUDE.md). IF no frontend files changed: SKIP,
  set {browser_result} = "N/A (no frontend changes)"

  TOOL DISCOVERY (CRITICAL):
    Use ToolSearch to find a browser-automation MCP (search "browser" or
    "playwright" or "chrome-devtools").
    IF tools not found:
      Set {browser_result} = "SKIPPED (browser-automation MCP not available)"
      PROCEED past this step

  PREREQUISITE: Check dev server at the project's documented dev URL
  (declared in CLAUDE.md, commonly http://localhost:3000).
    Use browser_navigate to the dev URL
    IF connection refused or timeout:
      Set {browser_result} = "SKIPPED (dev server not running — start it via
      the project's documented dev command)"
      PROCEED past this step

  AUTHENTICATE: Log in via the browser using test credentials, if the project
  declares any in CLAUDE.md (account types, credential env vars, login URL).
    Navigate to the login URL
    Use browser_fill_form with credentials from the documented env vars
    Click sign-in
    Wait for redirect to authenticated landing
    IF credentials not set or login fails: Document and continue (some flows
    can still be verified anonymously)

  EXECUTE: For each affected page:
    browser_navigate to URL
    browser_snapshot for accessibility tree
    browser_console_messages (check for errors)
    Execute user flows per acceptance criteria (browser_click, browser_type, browser_fill_form)
    browser_snapshot after interactions to verify state changes
    browser_network_requests to verify API calls (check for 4xx/5xx)
    browser_take_screenshot of key states

  STORE: {browser_result} (pass/fail with details)

IF any FAIL:
  ATTEMPT: Fix the failures (up to 3 attempts per failure)
  IF still failing:
    OUTPUT: "Test suite has failures after fixes:
    {list failures}

    Options:
    1. Continue trying to fix
    2. Commit with known failures (not recommended)
    3. Go back to /build for reimplementation

    Reply: fix / commit / rebuild"

    WAIT FOR USER RESPONSE
```

### Step 5.5: Generate Frontend Manual Test Plan

```
ACTION: Generate a per-issue manual test plan for the user to run in the browser
before acceptance-criteria check and commit. This removes the need for the user
to ask "what should I manually test?" at the end of every verify.

GATE: Only emit the full plan if frontend files changed.
  CHECK: `git status --porcelain` + Implementation Notes for any files under
         the project's documented frontend directory (declared in CLAUDE.md).
  IF no frontend files changed:
    OUTPUT: "Frontend Manual Test Plan: N/A (no frontend changes in this issue)."
    PROCEED to Step 6

ACCOUNT AUTO-PICK:
  IF the project's CLAUDE.md declares multiple test account types (e.g. roles,
  tiers, personas) along with rules for when each applies:
    SELECT the account type that best matches the changed surfaces, per the
    project's documented rule.
  ELSE:
    USE the default test account documented in CLAUDE.md.
  DO NOT prompt the user for account selection — auto-select per the rule above.

PAGES TO TEST:
  DERIVE from:
  1. Any new/modified page/route files in the frontend → map to URL routes
  2. For changed components, grep for parent routes that import them
  3. Any changed API route handlers → list the consuming page

  BUILD: {affected_routes} as a deduplicated list. URL base: the project's
  documented dev URL (commonly http://localhost:3000).

STEPS (one per acceptance criterion):
  READ: Original Requirements / Acceptance Criteria from the issue file.
  FOR EACH acceptance criterion:
    PHRASE as: "{N}. On {route}, {user action} → expect {visible outcome}."
    Include specific UI selectors/labels when known from the implementation.
  Keep steps concrete — reference actual button text, form field labels, toast
  copy, or URL changes that a human can observe, not internal state.

EDGE CASES (auto-included when applicable):
  IF translation/i18n message files changed:
    ADD: "Switch each supported locale; confirm no missing-key fallbacks or
         untranslated source-language strings on the affected pages."
  IF a list/grid/table component changed:
    ADD: "Empty state: test with an account/dataset that has zero items."
  IF a form or mutation changed:
    ADD: "Error state: trigger a server error (e.g. disconnect network mid-
         submit) and confirm the error toast + form stays interactive."
  IF layout/CSS/responsive classes changed:
    ADD: "Resize to mobile viewport (≤640px); confirm layout does not break."
  IF a subscription/billing/feature gate changed:
    ADD: "Test as a free-tier account; confirm gate renders and CTA works."

OUTPUT FORMAT:

"━━ FRONTEND MANUAL TEST PLAN ━━

Issue: #{issue_number} — {title}
Account: {auto-picked account label}
URL base: {project's documented dev URL}
Dev server: ensure the project's documented dev command is running

Steps:
{numbered steps generated per acceptance criterion}

Edge cases to try:
{bulleted edge cases that matched the rules above}

━━ END MANUAL TEST PLAN ━━

Run through this list in a browser before replying to the commit prompt in
Step 8. Report any issues and I will fix before committing."

PROCEED to Step 6.
```

### Step 6: Verify Acceptance Criteria

```
ACTION: Check each requirement from Original Requirements

FOR EACH requirement/acceptance criterion:
  VERIFY: Is it met by the implementation?
  EVIDENCE: How was it verified (test name, manual check, etc.)
  STORE: {criterion, met: true/false, evidence}

IF any criteria not met:
  OUTPUT: "Acceptance criteria not fully met:
  {list unmet criteria}

  Options:
  1. Proceed with known limitations (document them)
  2. Go back to fix

  Reply: proceed / fix"

  WAIT FOR USER RESPONSE
```

### Step 7: Append Verification Section

```
ACTION: Append to .claude/implementations/issue-{issue_number}.md

Update status:
**Status:** Reviewing -> Complete

APPEND:
---

## Verification & Commit (Claude /verify)

**Agent:** Claude
**Date:** {date}

### Issues Addressed

| Issue | Severity | Fix Applied |
| ----- | -------- | ----------- |
{For each issue from GPT review that was addressed}

### Issues Deferred

| Issue | Severity | Reason |
| ----- | -------- | ------ |
{For each issue that was skipped/deferred}

### Test Results

- **Backend tests:** {result} ({pass}/{total} tests)
- **Frontend tests:** {result} ({pass}/{total} tests)
- **Type check:** {result}
- **Linting:** {result}
- **Frontend (browser automation):** {browser_result}
- **Migration Application Check:** {migration_check_result}
  - If applied: list each migration version + name that was applied
  - If skipped: include the "Known Deploy Gap" subsection from Step 4.6
  - If N/A: "No migration files in this issue"

### Acceptance Criteria

| Criterion | Met? | Evidence |
| --------- | ---- | -------- |
{For each acceptance criterion}

### Commit

- **Branch:** {current branch}
- **Message:** [#{issue_number}] {title}
- **SHA:** {will be filled after commit}
```

### Step 8: Pre-Commit Review

```
OUTPUT:
"---
READY TO COMMIT - FINAL REVIEW
---

Issue: #{issue_number} - {title}

Validation Results:
- Tests: {test_results}
- Type check: {tsc_results}
- Lint: {lint_results}
- Migration check: {migration_check_summary} (e.g. "✅ {n} applied" / "⚠️ {n} skipped — manual apply required" / "N/A")
- Acceptance criteria: {X}/{Y} met

GPT Review Issues:
- Critical fixed: {count}
- Important fixed: {count}
- Minor fixed: {count}
- Deferred: {count}

Files to commit ({count}):
{list files with brief descriptions}

Commit Message:
'[#{issue_number}] {title}'

Proceed with commit? (yes/no)
---"

WAIT FOR USER RESPONSE:
IF "yes": PROCEED to Step 9
IF "no":
  OUTPUT: "Commit cancelled. No changes made."
  STOP
```

### Step 9: Git Commit

```
STEP 1: Stage files
EXECUTE: git add {list of changed files}

STEP 2: Create commit
COMMIT_MESSAGE: "[#{issue_number}] {title}"
EXECUTE: git commit -m "{COMMIT_MESSAGE}"

STEP 3: Get commit SHA
EXECUTE: git rev-parse HEAD
STORE: {commit_sha}

STEP 4: Update issue file with SHA
EDIT: Replace "SHA: {will be filled after commit}" with "SHA: {commit_sha}"

ERROR HANDLING:
IF commit fails:
  OUTPUT: "Git commit failed: {error}
  Check git status and resolve manually."
  STOP
```

### Step 10: Documentation Check

```
ACTION: Evaluate if this implementation requires documentation updates

The project's documentation set follows the structure declared in
`PROJECT_DOCUMENTATION_STANDARD.md` (see the kit's `shared-docs/`):

| Doc | Path | Scope |
| --- | ---- | ----- |
| CLAUDE.md | project root | Coding conventions, dev workflow, slash commands |
| AGENTS.md | project root | GPT hybrid workflow guide (architect & critic) |
| {ProjectName}.md | docs/context/ | Project manifest: tech stack, domain logic, architecture |
| business-plan.md | docs/ | Business context (revenue, GTM, regulatory) — if applicable |

STEP 10.1: Analyze Implementation Impact
---
READ: Implementation Notes, Architecture Plan, and changes from issue file

ASK YOURSELF for each doc:

**{ProjectName}.md** (project manifest) — update if:
- New table, service, or major component added
- Tech stack changed (new dependency, deprecated tool)
- Domain logic changed (new business rules, new user flows)
- Architecture changed (new data flow, new integration)
- API endpoints added or modified
- Test count significantly changed

**CLAUDE.md** (coding conventions) — update if:
- New coding pattern or convention established
- New slash command or workflow step added
- Dev workflow changed (new setup steps, new commands)
- File naming conventions changed
- New constraint doc added to .claude/docs/

**AGENTS.md** (GPT workflow guide) — update if:
- Hybrid workflow phases changed
- GPT's responsibilities changed
- New constraint docs added that GPT should read during review
- Handover protocol changed

EVALUATE:
IF YES to any question:
  Documentation update LIKELY NEEDED
  PROCEED to Step 10.2

IF NO to all questions:
  This is a minor change (bug fix, UI tweak, small refactor)
  LOG: "No documentation updates needed"
  PROCEED to Step 11
```

```
STEP 10.2: Prompt User for Documentation
---
OUTPUT TO USER:
"Documentation Check

This implementation may need documentation updates.

Changes detected:
{list relevant changes}

Docs to consider updating:
{list which of the 3 docs and why}

Would you like me to:
1. Draft documentation updates (recommended)
2. Skip documentation (you'll update later)

Reply: draft / skip"

WAIT FOR USER RESPONSE:
  IF "draft" or "yes" or "1":
    PROCEED to Step 10.3

  IF "skip" or "no" or "2":
    LOG: "User chose to skip documentation"
    PROCEED to Step 11
```

```
STEP 10.3: Draft and Apply Documentation Updates
---
FOR EACH doc that needs updating:
  READ: Current version of the doc
  DRAFT: Updated section(s) with new content
  PRESENT TO USER:
    "Proposed update to {doc_name}:

    [Show diff of proposed changes]

    Options:
    1. Accept and commit doc update
    2. Modify (you'll edit manually)
    3. Skip this doc

    Reply: accept / modify / skip"

  WAIT FOR USER RESPONSE

IF user accepts any drafts:
  APPLY: Changes to accepted docs
  git add {updated docs}
  git commit -m "[#{issue_number}] Update documentation for {feature}"
  LOG: "Documentation committed"

IF user chooses to modify/skip:
  LOG: "User will update documentation manually"
```

### Step 11: Post-Commit Actions

```
ACTION: Close GitHub issue (if GitHub enabled)

IF {github_enabled}:
  POST comment to issue with implementation summary
  CLOSE issue #{issue_number}

ACTION: Archive implementation file (MOVE, do NOT copy)
  IMPORTANT: Use `git mv` to move the file so git tracks the rename in a single operation.
  Do NOT use `cp` — the source file must be removed from implementations/.

  EXECUTE:
  mkdir -p .claude/implementations/archive/
  git mv .claude/implementations/issue-{issue_number}.md \
     .claude/implementations/archive/issue-{issue_number}.md
  git commit -m "chore: archive issue-{issue_number} implementation file"

OUTPUT:
"---
HYBRID WORKFLOW COMPLETE
---

Commit: {commit_sha}
Issue: #{issue_number} - closed
Archive: .claude/implementations/archive/issue-{issue_number}.md
{if docs updated: "Documentation: Updated ({list docs})"}

Workflow Summary:
Phase 1 (Scope):     Claude explored codebase
Phase 2 (Architect): GPT designed the plan
Phase 3 (Build):     Claude implemented the plan
Phase 4 (Review):    GPT reviewed the implementation
Phase 5 (Verify):    Claude fixed issues and committed

Next Steps:
1. Review commit: git show {commit_sha}
2. Push to remote: git push origin {current_branch}
---"
```

---

## Error Handling

### GPT Review References Wrong File/Line

```
IF GPT's review references a file:line that doesn't match:
  SEARCH: For the issue GPT described in nearby code
  IF found: Fix at correct location, note in Issues Addressed
  IF not found: Skip, document as "Could not locate issue"
```

### Fix Causes Cascade Failures

```
IF fixing one GPT issue breaks other tests:
  ASSESS: Is the fix correct but tests need updating?
  IF yes: Update the tests
  IF no: Revert the fix, document as deferred
```

### Migration Application Edge Cases

```
IF the project's list-applied command/MCP returns an error or times out:
  Treat as "tooling unavailable" — fall back to printing the manual reminder
  per Step 4.6, do not block the verify.

IF the issue's migration file uses a renamed/different version identifier
than the one currently on disk (e.g. attempt-2 renamed it to fix a collision):
  Use the CURRENT filename's identifier when comparing against applied versions.
  Old identifiers from prior attempts should be ignored.

IF the apply call succeeds but a follow-up read still returns "column/table
not found" or similar schema-cache errors:
  The framework's schema cache may be cold — wait a few seconds and re-check.
  If still missing, instruct the user to manually trigger a schema reload via
  the framework's documented mechanism (e.g. NOTIFY pgrst for PostgREST,
  client regenerate for Prisma, restart for cached ORMs).

IF the issue's migrations include both the issue's own NEW migrations AND
shared/coordinated migrations from a sibling issue (multiple parallel issues
touching the same tables):
  Only auto-apply migrations explicitly listed in THIS issue's Implementation
  Notes. Surface sibling-issue migration gaps as a warning, but do not apply
  them — that's the sibling issue's verify responsibility.
```

---

## What This Command Does NOT Do

- Push to remote (user does this manually)
- Re-run GPT's review (that's Phase 4)
- Make architectural changes (go back to /architect)
- Create pull requests
