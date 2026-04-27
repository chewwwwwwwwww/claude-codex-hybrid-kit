# Catchup Command

## Path Resolution (IMPORTANT)

All `.claude/` references in this command are **project-relative**, not global.

```
RESOLVE PATHS AS:
  .claude/implementations/  →  {current_project}/.claude/implementations/

EXAMPLE (working on acme-app):
  .claude/implementations/issue-123.md
  RESOLVES TO: acme-app/.claude/implementations/issue-123.md
  NOT: VS/.claude/implementations/issue-123.md

DETERMINE CURRENT PROJECT:
  - Check which project's CLAUDE.md was loaded
  - Use the conversation context to identify project
```

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

---

## Execution Instructions

When the user says "Catchup issue #123", "Status issue #123", "Where are we with #123" (or similar), execute these steps:

### Step 1: Parse Issue Number

Extract the issue number from the user's message.

- Valid formats: "issue #123", "#123", "Catchup 123", "status of 123"
- Store as: `{issue_number}`

### Step 2: Locate Markdown File

```
ACTION: Check if file exists: .claude/implementations/issue-{issue_number}.md
IF NOT EXISTS:
  OUTPUT: "No implementation found for issue #{issue_number}.

  Possible reasons:
  - Implementation not started yet (run 'Implement issue #{issue_number}')
  - Wrong issue number
  - File was deleted or moved

  Would you like to start implementation?"
  STOP

IF EXISTS: Proceed to Step 3
```

### Step 3: Read and Parse Markdown File

```
ACTION: Read entire contents of .claude/implementations/issue-{issue_number}.md

EXTRACT:
- Issue title (from header)
- Which agents have completed (look for "## Implementation Notes", "## Critic Review", "## Testing Results")
- Status of each agent (look for "**Status:**" lines)
- Current attempt numbers (if multiple attempts exist)
- Last action taken (most recent agent output)
- Any blockers or issues flagged

ANALYZE:
- Determine current stage: Implementation / Critic / Testing / Complete / Blocked
- Identify if paused at a checkpoint
- Count retry attempts
- Check if waiting for human decision
```

### Step 4: Determine Workflow State

```
LOGIC:
IF Testing Results exists with ✅ SUCCESS:
  state = "Ready to commit"

ELSE IF Testing Results exists with ⚠️ PARTIAL or ❌ BLOCKED:
  state = "Paused at Testing checkpoint"
  waiting_for = "Human decision on testing issues"

ELSE IF Critic Review exists with ⚠️ PARTIAL or ❌ BLOCKED:
  state = "Paused at Critic checkpoint"
  waiting_for = "Human decision on critic issues"

ELSE IF Security Review exists with ⚠️ SECURITY CONCERNS or ❌ SECURITY BLOCKED:
  state = "Paused at Security checkpoint"
  waiting_for = "Human decision on security issues"

ELSE IF Security Review exists with ✅ SECURITY APPROVED:
  state = "Between Security and Testing (in progress)"

ELSE IF Critic Review exists with ✅ SUCCESS:
  state = "Between Critic and Security/Testing (in progress)"

ELSE IF Implementation Notes exists with ❌ BLOCKED:
  state = "Blocked at Implementation"
  waiting_for = "Blocker resolution"

ELSE IF Implementation Notes exists with ✅ or ⚠️:
  state = "Between Implementation and Critic (in progress)"

ELSE:
  state = "Initialized but not started"
```

### Step 5: Format and Present Summary

```
OUTPUT FORMAT:

Issue #{issue_number}: {title}

Progress:
├─ {status_icon} Implementation (Attempt {N}) - {brief_status}
├─ {status_icon} Critic Review - {brief_status}
├─ {status_icon} Security Review - {brief_status} (if exists)
└─ {status_icon} Testing - {brief_status}

Current Stage: {state}
Last Action: {what_just_happened}

{IF waiting_for_decision:}
Waiting For: {what_decision_is_needed}

Quick Summary:
{2-3 sentence summary of current situation}

{IF issues_flagged:}
Issues Found:
- {issue_1}
- {issue_2}

Next Steps:
{specific actionable commands the user can run}

{IF want_details:}
Full details: .claude/implementations/issue-{issue_number}.md
```

### Step 6: Add Contextual Recommendations

```
BASED ON STATE, ADD RECOMMENDATIONS:

IF state == "Ready to commit":
  "Run: Commit issue #{issue_number}"

IF state == "Paused at Critic checkpoint":
  "Options: 'proceed to testing' / 'retry implementation' / 'stop'"

IF state == "Paused at Testing checkpoint" AND lint warnings:
  "Options: 'proceed to commit' / 'auto-fix linting' / 'retry implementation'"

IF state == "Paused at Testing checkpoint" AND test failures:
  "Options: 'retry implementation' / 'let me investigate'"

IF state == "Blocked":
  "Resolve blocker, then: 'retry [agent]' or 'continue workflow'"

IF retry_attempts > 2:
  "Note: {retry_attempts} attempts made. Consider if approach needs rethinking."
```

---

## Example Outputs

### Example 1: Paused at Critic Checkpoint

```
Issue #123: Add chord transposition feature

Progress:
├─ ✅ Implementation complete (Attempt 1)
├─ ⚠️ Critic found important issues
└─ ⏸️ Testing not started

Current Stage: Paused at Critic checkpoint
Last Action: Critic completed review

Waiting For: Your decision on Critic's findings

Quick Summary:
Implementation is complete. Critic found 2 important issues: missing input validation and performance concern with large datasets. Workflow paused for your review.

Issues Found:
- Missing validation for invalid key signatures (🟡 Important)
- Performance concern when transposing large songs (🟡 Important)

Next Steps:
Choose one:
- 'proceed to testing' - Accept issues and continue
- 'retry implementation' - Fix issues first
- Read details: .claude/implementations/issue-123.md
```

### Example 2: Ready to Commit

```
Issue #247: Implement drag-and-drop positioning

Progress:
├─ ✅ Implementation complete (Attempt 1)
├─ ✅ Critic complete - no significant issues
└─ ✅ Testing complete - all passing

Current Stage: Ready to commit
Last Action: Testing validated everything successfully

Quick Summary:
All stages complete. Tests passing (25/25), no lint warnings, all acceptance criteria met. Implementation is ready for finalization.

Next Steps:
Run: Commit issue #247
```

### Example 3: Blocked State

```
Issue #456: Add PDF export feature

Progress:
├─ ❌ Implementation blocked
└─ ⏸️ Workflow stopped

Current Stage: Blocked at Implementation
Last Action: Implementation Agent encountered blocker

Waiting For: Blocker resolution

Quick Summary:
Implementation cannot proceed due to missing pdf-lib dependency. Install the required package to continue.

Blocker:
Cannot import 'pdf-lib' - package not found

Next Steps:
1. Run: npm install pdf-lib
2. Then: retry implementation

Full details: .claude/implementations/issue-456.md
```

---

## Status Icons Reference

```
✅ = Successfully completed
⚠️ = Completed with warnings/concerns
❌ = Failed or blocked
⏸️ = Not started / paused
🔄 = In progress
```

---

## Error Handling

**If markdown file is corrupted or unreadable:**

```
OUTPUT: "Cannot read workflow state for issue #{issue_number}.

The file exists but appears corrupted or improperly formatted.
Location: .claude/implementations/issue-{issue_number}.md

Please review the file manually or restart the implementation."
STOP
```

**If multiple attempt sections found but unclear which is latest:**

```
Look for highest attempt number
Use most recent timestamp if available
Default to last section in file
```

---

## Performance Note

This command is READ-ONLY:

- Does not invoke any agents
- Does not modify any files
- Does not change workflow state
- Simply analyzes and reports

Safe to run repeatedly without side effects.