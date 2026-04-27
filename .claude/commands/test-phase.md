# Test Phase Command

## Path Resolution (IMPORTANT)

This command operates on the CURRENT PROJECT, not global .claude/.

When reading/writing files:

- `.claude/implementations/issue-123.md` = `{project}/.claude/implementations/issue-123.md`
- NOT `~/.claude/implementations/issue-123.md`

## GitHub Repository Resolution (CRITICAL)

**Before any GitHub operations, determine the correct repository:**

1. **Check CLAUDE.md** (in project root):
   - Look for line starting with `**Repository:**`
   - Example: `**Repository:** acme-org/acme-app`
   - This is the PREFERRED method - project explicitly declares its repo

2. **Fall back to git remote**:
   - Run: `git remote get-url origin`
   - Parse: `git@github.com:owner/repo.git` → `owner/repo`
   - Or: `https://github.com/owner/repo.git` → `owner/repo`

3. **Store as {github_repo}** for all GitHub operations

**NEVER hardcode a repository name. Always resolve dynamically.**

## Purpose

Runs ONLY the Testing Agent in a fresh session. Use after Critic approves implementation.

## When to Use

- After `/critic-phase` says ✅ SUCCESS or acceptable ⚠️ PARTIAL
- Part of manual multi-session workflow
- When you want fresh context for testing

---

## Execution Instructions

### Step 1: Parse Issue Number

```
EXTRACT: {issue_number} from command
Format: /test-phase #123
```

### Step 2: Verify Prerequisites

```
CHECK: .claude/implementations/issue-{issue_number}.md exists
CHECK: Has "## Implementation Notes" section
CHECK: Has "## Critic Review" section

IF MISSING:
  ERROR: "Run /implement-phase and /critic-phase first."
  STOP
```

### Step 3: Execute Testing Agent in Phase Mode

```
FILE: .claude/agents/04-testing-agent.md

INSTRUCTION: "You are in PHASE MODE.
Load context from .claude/implementations/issue-{issue_number}.md
Follow Phase Mode instructions.
Run all validations, write results, report completion.

After automated tests complete and before you emit the final status report,
ALSO run Step 3.5 (Frontend Manual Test Plan) below and print it as part of
your output."
```

### Step 3.5: Frontend Manual Test Plan (runs inside Testing Agent)

```
ACTION: Generate a per-issue manual test plan for the user to run in the browser
before replying with ✅/⚠️/❌. This removes the need for the user to ask
"what should I manually test?" at the end of every test-phase.

GATE: Only emit the full plan if frontend files changed.
  CHECK: `git status --porcelain` + Implementation Notes for any files under
         the project's documented frontend directory (declared in CLAUDE.md).
  IF no frontend files changed:
    OUTPUT: "Frontend Manual Test Plan: N/A (no frontend changes in this issue)."
    PROCEED to status report.

ACCOUNT AUTO-PICK:
  IF the project's CLAUDE.md declares multiple test account types (e.g. roles,
  tiers, personas) along with rules for when each applies:
    SELECT the account type that best matches the changed surfaces.
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

Run through this list in a browser before replying ✅/⚠️/❌."

THEN: Proceed to the final ✅/⚠️/❌ status report.
```

---

## Testing Agent Will Execute

1. Load context from markdown
2. Run code quality validation (Prettier, ESLint, TypeScript)
3. Run automated tests
4. Verify acceptance criteria
5. Generate and print the Frontend Manual Test Plan (Step 3.5) if frontend files changed
6. Write comprehensive results to markdown
7. Report status and next steps

---

## Status Meanings

**✅ SUCCESS (no warnings):**

- All tests pass
- No lint errors
- All acceptance criteria met
- **Next:** `/commit-phase #123`

**⚠️ PARTIAL (with warnings):**

- Tests pass BUT lint warnings exist
- **Options:**
  - Proceed: `/commit-phase #123`
  - Auto-fix: Re-run with "auto-fix linting"
  - Manual fix: `/implement-phase #123` (retry)

**❌ BLOCKED (tests failed):**

- Tests failing or acceptance criteria not met
- **Next:** `/implement-phase #123` (retry with test failure details)

---

## Usage Examples

### Example 1: Tests Pass

```
User: "/test-phase #123"
[Testing runs]
Output: "✅ SUCCESS - All 25 tests passing, no warnings"
User: Closes VS Code
Next: /commit-phase #123
```

### Example 2: Lint Warnings

```
User: "/test-phase #123"
[Testing runs]
Output: "⚠️ PARTIAL - Tests pass, 3 ESLint warnings"
User decides: proceed or fix
User: Closes VS Code
```

### Example 3: Tests Fail

```
User: "/test-phase #123"
[Testing runs]
Output: "❌ BLOCKED - 2 tests failing"
User: Closes VS Code
Next: /implement-phase #123 (retry to fix)
```

### Example 4: Auto-Fix Linting

```
User: "/test-phase #123"
[Testing runs]
Output: "⚠️ PARTIAL - 5 ESLint warnings"

User: "Auto-fix linting and re-test"
[Testing runs eslint --fix]
[Re-runs tests]
Output: "✅ SUCCESS - Warnings fixed, all tests pass"
```

---

## Integration with Workflow

**Position:**

```
Session 1: /implement-phase #123 → Close
Session 2: /critic-phase #123 → Close
Session 3: /test-phase #123 → Close ← YOU ARE HERE
Session 4: [Based on result]
  IF ✅: /commit-phase #123
  IF ⚠️: Decide proceed/auto-fix/retry
  IF ❌: /implement-phase #123 (retry)
```

---

## Error Handling

**Missing Implementation:**

```
"❌ No implementation to test.
Run /implement-phase #123 first."
```

**Missing Critic Review:**

```
"⚠️ No Critic review found.
Proceeding with testing, but consider running /critic-phase first."
[Continues anyway - testing can happen without Critic]
```

---

**Remember:** Testing is the final validation before commit. After this phase completes successfully, you're ready to finalize with `/commit-phase`.
