# Commit Command

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

## Execution Instructions

When the user says "Commit issue #123" (or similar), execute these steps:

### Step 1: Parse Issue Number

Extract the issue number from the user's message.

- Valid formats: "Commit issue #123", "Commit #123", "Finalize 123"
- Store as: `{issue_number}`

### Step 2: Verify Markdown File Exists

```
ACTION: Check if file exists: .claude/implementations/issue-{issue_number}.md
IF NOT EXISTS:
  OUTPUT: "No implementation found for issue #{issue_number}.
  Run 'Implement issue #{issue_number}' first."
  STOP
```

### Step 3: Read Workflow State

```
ACTION: Read .claude/implementations/issue-{issue_number}.md

EXTRACT:
- Issue title
- Testing Agent's final status
- All files that were changed
- Acceptance criteria and their verification status
- Implementation summary
- Key decisions made
```

### Step 4: Phase 1 - Final Validation (Read-Only)

#### Check 1: Verify Security Approval (if Security ran)

```
VALIDATION: Look for "## [SECURITY]" section in markdown

IF Security Review exists:
  CHECK: Status should be "✅ SECURITY APPROVED"

  IF status is "⚠️ SECURITY CONCERNS":
    EXTRACT: What security concerns exist
    PAUSE and OUTPUT:
      "⚠️ Security review completed with concerns:
      {list security concerns}

      Options:
      1. 'proceed anyway' - Commit with these security concerns
      2. 'fix security first' - Go back and address them

      ⚠️ WARNING: Proceeding with security concerns is risky.
      Recommendation: Fix security issues before committing.
      Reply: proceed / fix"

    WAIT FOR USER RESPONSE:
      IF "proceed" or "proceed anyway": Continue to Check 2
      IF "fix" or other: STOP with message "Address security concerns and re-run"

  IF status is "❌ SECURITY BLOCKED":
    OUTPUT: "❌ Cannot commit: Critical security vulnerabilities found.
    {show vulnerability details}

    Fix the security issues before committing."
    STOP

IF Security Review does NOT exist:
  LOG: "No security review found (likely low-risk changes)"
  Continue to Check 2
```

#### Check 2: Verify Testing Complete

```
VALIDATION: Look for "## Testing Results" section
CHECK: Status should be "✅ SUCCESS"

IF status is "⚠️ PARTIAL":
  EXTRACT: What warnings exist (lint warnings, etc.)
  PAUSE and OUTPUT:
    "⚠️ Testing completed with warnings:
    {list warnings}

    Options:
    1. 'proceed anyway' - Commit with these warnings
    2. 'fix warnings first' - Go back and address them

    Recommendation: Fix warnings before committing.
    Reply: proceed / fix"

  WAIT FOR USER RESPONSE:
    IF "proceed" or "proceed anyway": Continue to Check 2
    IF "fix" or other: STOP with message "Address warnings and re-run Testing Agent"

IF status is "❌ BLOCKED":
  OUTPUT: "❌ Cannot commit: Testing failed or blocked.
  {show failure details}

  Fix the issues and re-run Testing Agent before committing."
  STOP
```

#### Check 3: TypeScript Type Check (if applicable)

```
ACTION: Check if tsconfig.json exists

IF EXISTS:
  EXECUTE: tsc --noEmit

  IF type errors found:
    OUTPUT: "❌ Type errors detected:
    {list errors with file:line}

    Cannot commit with type errors.
    Fix these and re-run Testing Agent."
    STOP

  IF passes:
    LOG: "✅ TypeScript type check passed"

IF NOT EXISTS:
  LOG: "⚪ TypeScript not configured - skipping type check"
```

#### Check 3: Verify Acceptance Criteria

```
ACTION: Read Testing Agent's "Acceptance Criteria Verification" section

FOR EACH criterion:
  IF marked as "❌ Not met":
    COLLECT: Which criteria are not met

IF any criteria not met:
  OUTPUT: "⚠️ Acceptance criteria not fully met:
  {list unmet criteria}

  Options:
  1. 'proceed anyway' - Document as known limitations
  2. 'go back' - Fix criteria before committing

  Reply: proceed / go back"

  WAIT FOR USER RESPONSE:
    IF "proceed": Continue with note to document limitations
    IF other: STOP
```

#### Check 4: Verify Uncommitted Changes

```
EXECUTE: git status --porcelain

IF no output (no changes):
  OUTPUT: "No changes to commit. Everything is already committed or working directory is clean."
  STOP

ELSE:
  EXTRACT: List of modified/new/deleted files
  LOG: "Changes detected: {file_count} files"
```

### Step 5: Phase 2 - Pre-Commit Review (Human Checkpoint)

```
BEFORE printing the pre-commit block, generate a Frontend Manual Test Plan
and embed it INSIDE the pre-commit output. This removes the need for the user
to ask "what should I manually test?" before approving the commit.

FRONTEND MANUAL TEST PLAN (generation rules):

GATE: Only emit the full plan if frontend files changed.
  CHECK: `git status --porcelain` + Implementation Notes for any files under
         the project's documented frontend directory (declared in CLAUDE.md).
  IF no frontend files changed:
    Set {manual_test_plan_block} = "Frontend Manual Test Plan: N/A (no frontend changes)."
    SKIP the rest of generation, use that string in the output.

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
  READ: Original Requirements / Acceptance Criteria from the markdown file.
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

ASSEMBLE {manual_test_plan_block} as:
"━━ FRONTEND MANUAL TEST PLAN ━━

Account: {auto-picked account label}
URL base: {project's documented dev URL}
Dev server: ensure the project's documented dev command is running

Steps:
{numbered steps generated per acceptance criterion}

Edge cases to try:
{bulleted edge cases that matched the rules above}

━━ END MANUAL TEST PLAN ━━"

---

OUTPUT FORMAT (now embed the plan in the review):

"━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
READY TO COMMIT - FINAL REVIEW
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Issue: #{issue_number} - {title}

Validation Results:
✅ Testing: {test_results}
✅ Type check: {typescript_results}
✅ Acceptance criteria: {criteria_results}

Files to be committed ({count}):
{list each file with brief description from implementation notes}

Commit Message:
'[#{issue_number}] {title}'

{manual_test_plan_block}

Run through the manual test plan in a browser before replying 'yes'. Report
any issues and I will fix before committing.

Actions that will be taken:
1. Document complete implementation in GitHub issue #{issue_number}
2. Stage and commit all changes
3. Close GitHub issue #{issue_number}
4. (You will push manually afterwards)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Proceed with commit? (yes/no)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

WAIT FOR USER RESPONSE:
IF "yes": Continue to Phase 3
IF "no" or other:
  OUTPUT: "Commit cancelled. No changes made."
  STOP
```

### Step 6: Phase 3 - Document in GitHub Issue

```
ACTION: Prepare documentation for GitHub issue

DOCUMENTATION FORMAT:
---
## ✅ Implementation Complete

**Summary:**
{2-3 paragraph summary from Implementation Agent notes}

**Files Changed:**
{for each file:}
- `{filepath}` - {description from implementation notes}

**Testing Results:**
- ✅ Tests: {X}/{Y} passing
- ✅ Type check: Passed (or "Not applicable")
- ✅ Linting: Clean (or "{X} warnings accepted")
- ✅ Acceptance criteria: {X}/{Y} met

**Key Decisions:**
{list important decisions from Implementation notes}

**Acceptance Criteria:**
{for each criterion:}
✅ **AC{N}:** {criterion text}
   - {verification details from Testing Agent}

{if any not fully met:}
**Known Limitations:**
- {limitation 1}
- {limitation 2}

**Implementation Details:**
Complete implementation history: `.claude/implementations/issue-{issue_number}.md`

**Commit:** {will be added after committing}
---

ACTION: Post this as a comment on GitHub issue #{issue_number}
TOOL: Use GitHub MCP or API
ERROR HANDLING: If API fails, log error but continue (can update manually)
```

### Step 7: Phase 4 - Git Commit

```
STEP 1: Stage all changed files
EXECUTE: git add {list of files from git status}

STEP 2: Create commit
COMMIT_MESSAGE: "[#{issue_number}] {issue_title}"
EXECUTE: git commit -m "{COMMIT_MESSAGE}"

STEP 3: Get commit SHA
EXECUTE: git rev-parse HEAD
STORE: {commit_sha}

ERROR HANDLING:
IF commit fails:
  OUTPUT: "❌ Git commit failed: {error message}

  Possible issues:
  - Nothing staged (shouldn't happen if validation passed)
  - Git config issues
  - Repository state problems

  Check git status manually."
  STOP (do not close GitHub issue)
```

### Step 8: Phase 5 - Close GitHub Issue

```
ACTION: Update previous GitHub comment with commit SHA
EDIT_COMMENT: Add line "**Commit:** {commit_sha}" to the documentation

ACTION: Close the issue
TOOL: GitHub MCP or API - close issue #{issue_number}

ERROR HANDLING:
IF closing fails:
  OUTPUT: "⚠️ Issue #{issue_number} could not be auto-closed.
  Commit successful ({commit_sha}) but please close the issue manually."
  Continue to Phase 6
```

### Step 9: Phase 6 - Documentation Check

```
ACTION: Evaluate if this implementation requires documentation updates

The project has three documentation files with distinct scopes:

| Doc | Path | Scope |
| --- | ---- | ----- |
| CLAUDE.md | project root | Coding conventions, dev workflow, slash commands |
| AGENTS.md | project root | GPT hybrid workflow guide (architect & critic) |
| {ProjectName}.md | docs/context/ | Project manifest: tech stack, domain logic, architecture |

Additionally, constraint docs live in .claude/docs/0X-*.md.

STEP 9.1: Analyze Implementation Impact
---
READ: Implementation Notes and changes from markdown file

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

**Constraint docs** (.claude/docs/) — update if:
- New rules that future implementations must follow
- Changes to existing constraint behavior
- New failure modes to avoid

EVALUATE:
IF YES to any question:
  Documentation update LIKELY NEEDED
  PROCEED to Step 9.2

IF NO to all questions:
  This is a minor change (bug fix, UI tweak, small refactor)
  SKIP documentation check
  PROCEED to Step 10
```

```
STEP 9.2: Prompt User for Documentation
---
OUTPUT TO USER:
"📝 Documentation Check

This implementation may need documentation updates.

Changes detected:
{list relevant changes}

Docs to consider updating:
{list which docs and why, e.g.:
- docs/context/{ProjectName}.md: New flashcard_lesson_links table added
- CLAUDE.md: New junction table pattern established
- .claude/docs/06-data-isolation-rls.md: New RLS policies added
}

Would you like me to:
1. Draft documentation updates (recommended)
2. Skip documentation (you'll update later)

Reply: draft / skip"

WAIT FOR USER RESPONSE:
  IF "draft" or "yes" or "1":
    PROCEED to Step 9.3

  IF "skip" or "no" or "2":
    LOG: "User chose to skip documentation"
    PROCEED to Step 10
```

```
STEP 9.3: Draft and Apply Documentation Updates
---
FOR EACH doc that needs updating:
  READ: Current version of the doc
  DRAFT: Updated section(s) with new content
  PRESENT TO USER:
    "📄 Proposed update to {doc_name}:

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

### Step 10: Phase 7 - CLAUDE.md Health Check

```
ACTION: Check CLAUDE.md size and health (independent of content updates above)

READ: Current CLAUDE.md file

MEASURE BLOAT:
- Line count of CLAUDE.md

BLOAT THRESHOLDS:
- 🟢 Healthy: <800 lines
- 🟡 Warning: 800-1200 lines
- 🔴 Critical: >1200 lines

IF 🟢 Healthy:
  LOG: "CLAUDE.md is healthy ({line_count} lines)"
  PROCEED to Step 10.2

IF 🟡 Warning or 🔴 Critical:
  OUTPUT: "📝 CLAUDE.md is {line_count} lines [{🟡/🔴} {status}]
  Consider archiving to .claude/claude-versions/ to keep it lean.

  Options:
  1. Archive + refresh (move current, create fresh)
  2. Skip (deal with it later)

  Reply: 1 / 2"

  WAIT FOR USER RESPONSE:
    IF "1": Archive current CLAUDE.md and create fresh version
    IF "2": Continue
```

```
STEP 10.2: Archive Implementation File (MANDATORY)
---
ACTION: Move completed implementation to archive

NOTE: .claude/ is gitignored, so use plain `mv` (NOT `git mv`).
This step is MANDATORY after a successful commit — do NOT skip it.

EXECUTE:
  mkdir -p .claude/implementations/archive/
  mv .claude/implementations/issue-{issue_number}.md \
     .claude/implementations/archive/issue-{issue_number}.md

VERIFY: Confirm the source file no longer exists in implementations/
  ls .claude/implementations/issue-{issue_number}.md
  (should return "No such file or directory")

OUTPUT: "Archived issue-{issue_number}.md to .claude/implementations/archive/"
```

```
STEP 10.3: Final Report
---
OUTPUT:
"━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ COMMIT COMPLETE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Commit: {commit_sha}
Issue: #{issue_number} closed
Documentation: Updated in GitHub issue
Implementation: Archived to .claude/implementations/archive/
{if docs updated: "Documentation: Updated ({list updated docs})"}

Next Steps:
1. Review commit: git show {commit_sha}
2. Push to remote: git push origin {current_branch}

Great work! 🎉
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

---

## Edge Case Handling

### Edge Case 1: User Wants Custom Commit Message

```
IF user says: "Commit issue #123 with message 'feat: add transposition'"

EXTRACT: Custom message portion
BUILD_MESSAGE: "[#123] feat: add transposition"

INFORM USER:
"Using commit message: '[#123] feat: add transposition'
Issue reference [#123] added automatically.

Proceed? (yes/no)"
```

### Edge Case 2: Proceeding with Warnings

```
IF user chose "proceed anyway" after warnings:

ADD TO GITHUB DOCUMENTATION:
"**Note:** This implementation was committed with the following accepted warnings:
- {warning 1}
- {warning 2}

These were reviewed and deemed acceptable for this release."
```

### Edge Case 3: Git Commit Fails

```
IF git commit returns error:

DO NOT CLOSE GITHUB ISSUE
DO NOT UPDATE GITHUB DOCUMENTATION

OUTPUT: "❌ Commit failed: {error}

No changes made to:
- GitHub issue (still open)
- GitHub documentation (not posted)

Please resolve the git issue and try again."

STOP
```

### Edge Case 4: GitHub API Unavailable

```
IF GitHub operations fail:

STILL COMMIT TO GIT (local commit is independent)

OUTPUT: "⚠️ Git commit successful: {commit_sha}

However, GitHub updates failed:
- Could not post documentation
- Could not close issue

Please update manually:
1. Add commit SHA to issue: {commit_sha}
2. Close issue #{issue_number}

Or retry: 'Update GitHub for issue #{issue_number}'"
```

---

## What This Command Does NOT Do

Does NOT:

- ❌ Modify code (all code changes should be done before this)
- ❌ Run tests (Testing Agent already did this)
- ❌ Auto-fix anything (should be handled in Testing phase)
- ❌ Push to remote (user does this manually)
- ❌ Create pull requests
- ❌ Run build scripts or deploy

Does:

- ✅ Validate everything is ready
- ✅ Document thoroughly in GitHub
- ✅ Commit locally with proper message
- ✅ Close GitHub issue
- ✅ Preserve implementation history

---

## Validation Summary

Before committing, this command verifies:

1. ✅ Testing completed successfully (or warnings accepted)
2. ✅ TypeScript type check passes (if applicable)
3. ✅ Acceptance criteria met (or limitations documented)
4. ✅ There are actually changes to commit
5. ✅ Human explicitly confirms "yes"

Only after ALL validations pass does it commit and close the issue.
