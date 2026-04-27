# Build Command (Hybrid Phase 3)

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

Hybrid Phase 3: Implementation. Reads GPT's architecture plan and implements the code, writing tests alongside.

**What this command does:**

- Reads the architecture plan from GPT
- Implements code following GPT's plan + CLAUDE.md conventions
- Writes tests alongside implementation
- Appends Implementation Notes section to the issue file

**What this command does NOT do:**

- Deviate from GPT's plan without documenting why
- Commit code (that's /verify's job)
- Run the full test suite for final validation (that's /verify's job)
- Skip writing tests

---

## Execution Instructions

When the user says "/build #N" or "Build issue #N", execute these steps:

### Step 1: Parse Issue Number

```
EXTRACT: Issue number from user's message
FORMAT: "#N", "issue N", "build N"
STORE: {issue_number}
```

### Step 2: Read and Validate Issue File

```
ACTION: Read .claude/implementations/issue-{issue_number}.md

VALIDATE:
1. File exists
   IF NOT: OUTPUT "No issue file found for #{issue_number}. Run '/scope' first." STOP

2. Mode is "Hybrid"
   IF NOT: OUTPUT "Issue #{issue_number} is not in Hybrid mode. Use '/implement' for single-model." STOP

3. "## Architecture Plan" section exists
   IF NOT: OUTPUT "No architecture plan found. GPT needs to run '/architect #{issue_number}' first." STOP

EXTRACT:
- Original requirements
- Architecture plan (GPT's full plan)
- Codebase exploration notes (file references, patterns)
- Any open questions from the plan

IF open questions exist and are unresolved:
  OUTPUT: "The architecture plan has open questions:
  {list questions}

  Should I:
  1. Proceed with my best judgment on these
  2. Wait for clarification

  Reply: proceed / wait"

  WAIT FOR USER RESPONSE
```

### Step 3: Read Reference Files

```
ACTION: Read coding conventions and standards

READ: CLAUDE.md (project conventions, patterns, file naming)
READ: `~/.claude/CLAUDE.md` (your global cross-project conventions, if any)

READ: All source files referenced in GPT's architecture plan
- Read each file mentioned in "Implementation Steps"
- Understand current state before modifying
```

### Step 4: Determine Attempt Number

```
ACTION: Check issue file for existing Implementation Notes sections

COUNT: Number of "## Implementation Notes - Attempt" sections
STORE: {attempt_number} = count + 1

IF attempt_number > 1:
  READ: Previous attempt's notes
  READ: GPT Review that prompted the re-build (if exists)
  NOTE: Issues to address from previous attempt
```

### Step 5: Implement

```
ACTION: Follow GPT's architecture plan step by step

FOR EACH step in the architecture plan:

  5a. Read the target file (if modifying existing)
  5b. Implement the change as specified
  5c. Write tests for the new/changed code
  5d. Track what was done

IMPLEMENTATION RULES:
- Follow GPT's plan as closely as possible
- Use the project's CLAUDE.md and (if you maintain one) your global
  `~/.claude/CLAUDE.md` for coding conventions, file naming, and standards
  for error handling, security, and testability.
- Write tests alongside each piece of implementation, not deferred
- If deviating from GPT's plan, document the reason clearly

SECURITY-AWARE IMPLEMENTATION RULES (stack-agnostic baseline; project may
extend these in CLAUDE.md):
- NEVER store sensitive fields (subscription_status, rate_limits, credits,
  role, is_admin, is_premium) on records the end user can write directly.
  Put them on records with read-only access from the user, or behind
  server-side mutation paths.
- NEVER call AI providers, payment-provider secret APIs, or transactional
  email services from client-side code. Route everything through a
  server-side handler (API route, edge function, or backend service).
- NEVER put secret keys in client-side environment variables. Treat any
  variable with a publicly-exposed prefix (e.g. `NEXT_PUBLIC_`, `VITE_`,
  `EXPO_PUBLIC_`, `REACT_APP_`, `PUBLIC_`) as readable by every visitor.
- ALWAYS add server-side rate limiting on new API endpoints, especially AI
  and payment endpoints. Rate-limit counters must live where the user
  cannot tamper with them.
- ALWAYS use admin/service-tier database credentials only in server-side
  code paths.
- ALWAYS verify webhook signatures using the raw request body, not a parsed
  or re-serialized version. (Provider-specific signing details belong in
  the project's AGENTS.md or constraint docs.)

DEVIATION PROTOCOL:
IF a step in GPT's plan won't work (wrong assumption, missing dependency, etc.):
  1. Document why in Key Decisions
  2. Implement the closest viable alternative
  3. Flag it in "For Reviewer" section
  DO NOT silently change the approach

TRACK:
- Files created (path, description)
- Files modified (path, what changed)
- Tests written (file, count, what they cover)
- Key decisions made
- Deviations from plan
```

### Step 6: Run Quick Validation

```
ACTION: Run basic checks on the implementation (not full suite). Use the
project's documented test/typecheck/lint commands from CLAUDE.md.

IF a typed-language type-check command applies (e.g. `tsc --noEmit`, `mypy`):
  EXECUTE the project's documented type-check command, scoped to the relevant
  package/dir if the project is a monorepo
  IF errors: Fix them before proceeding

IF backend test files were written:
  EXECUTE the project's documented backend test command, targeted at the
  specific test files written (not the full suite)
  IF failures: Fix them before proceeding

IF frontend tests were written:
  EXECUTE the project's documented frontend test command, targeted at the
  specific test files written (not the full suite)
  IF failures: Fix them before proceeding

NOTE: Full test suite runs in /verify. This is just a smoke check.
```

### Step 7: Append Implementation Notes

```
ACTION: Append to .claude/implementations/issue-{issue_number}.md

Update status:
**Status:** Implementing -> Reviewing

APPEND:
---

## Implementation Notes - Attempt {attempt_number} (Claude /build)

**Agent:** Claude
**Date:** {date}
**Status:** {SUCCESS | PARTIAL | BLOCKED}

### Files Changed

| File | Action | Description |
| ---- | ------ | ----------- |
{For each file created/modified/deleted}

### Tests Written

| Test File | Count | Coverage Area |
| --------- | ----- | ------------- |
{For each test file}

### Key Decisions

{Decisions made during implementation and their rationale.
Especially document any deviations from GPT's plan.}

1. **{Decision}** - {Rationale}
   {If deviation: "Deviation from plan: GPT suggested X, implemented Y because Z"}

### For Reviewer

{Specific areas that need GPT's review attention}

- {Area 1}: {Why it needs attention}
- {Area 2}: {Why it needs attention}

### Security-Sensitive Areas (Auto-Flagged)

{Auto-flag any of the following if they were part of this implementation:}
- [ ] New/modified RLS policies: {table names and policy descriptions}
- [ ] New API endpoints: {paths — reviewer should verify auth + rate limiting}
- [ ] New environment variables: {names — reviewer should verify client vs server placement}
- [ ] AI/LLM integration: {what was added — reviewer should verify server-side only}
- [ ] Payment/billing changes: {what changed — reviewer should verify webhook sigs + server-side validation}
- [ ] Database schema changes: {tables added/modified — reviewer should verify RLS + sensitive field placement}
{If none of the above apply, state "No security-sensitive areas identified."}

---
*Next step: Open GPT (Codex) and run `/review {issue_number}` for critic & security review.*
---
```

### Step 8: Report Completion

```
OUTPUT:
"---
HYBRID PHASE 3: BUILD COMPLETE
---

Issue: #{issue_number} - {title}
Status: {SUCCESS | PARTIAL | BLOCKED}
Attempt: {attempt_number}

Changes:
- {count} files modified/created
- {count} tests written
{If deviations: "- {count} deviations from plan (documented)"}

Quick Validation:
- TypeScript: {PASS | N/A}
- Tests: {X}/{Y} passing
- Lint: {PASS | warnings}

{IF STATUS == BLOCKED:}
Blocker: {description}
Cannot proceed until: {what needs to happen}

{IF STATUS == SUCCESS or PARTIAL:}
Next Step:
Open GPT (Codex) and run:
  /review {issue_number}

GPT will review the implementation, check security,
and write the GPT Review section.
---"
```

---

## Error Handling

### Architecture Plan References Non-Existent File

```
IF GPT's plan references a file that doesn't exist:
  CHECK: Is it a file GPT expects us to CREATE?
  IF yes: Create it as specified
  IF no: Document as deviation, implement closest alternative
```

### Tests Fail During Implementation

```
IF tests fail during Step 6:
  ATTEMPT: Fix the failing tests (up to 3 attempts)
  IF still failing after 3 attempts:
    STATUS = PARTIAL
    Document which tests fail and why in Implementation Notes
    Proceed to Step 7 (GPT may have insights)
```

### Plan Step Is Ambiguous

```
IF a plan step is unclear:
  CHECK: Can Codebase Exploration section clarify?
  CHECK: Can the existing code patterns clarify?
  IF still unclear:
    Make best judgment call
    Document in Key Decisions: "Interpreted '{ambiguous instruction}' as {interpretation}"
    Flag in For Reviewer
```

---

## What This Command Does NOT Do

- Create architecture plans (GPT does this)
- Run the full test suite (that's /verify)
- Commit or push code (that's /verify)
- Re-run GPT's review
- Modify the Architecture Plan section
