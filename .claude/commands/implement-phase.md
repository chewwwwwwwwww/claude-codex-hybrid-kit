# Implement Phase Command

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
This command runs the Implementation Agent in a fresh session as part of a manual multi-session workflow. It can BOTH initialize new implementations AND retry existing ones.

**CRITICAL: For new implementations, use this in a FRESH session after planning.**

User has THREE options for providing the plan:

### Option 1: Reference Plan File (EASIEST ✅)
```
User: "/implement-phase - use plan from .claude/plan-2026-01-14.md"
Command: Reads file, creates issue, implements
```

### Option 2: Paste Plan in Message
```
User: [Pastes plan]
User: "/implement-phase"
Command: Extracts from message, creates issue, implements
```

### Option 3: Legacy (Same Session)
```
User: "/implement-phase"
Command: Searches context (may be bloated)
```

**Recommended workflow:**
1. Complete planning in Session 1
2. Note the plan file path (if saved)
3. Close VS Code (end planning session)
4. Open fresh VS Code (Session 2)
5. Reference file OR paste plan
6. Run `/implement-phase`

This ensures implementation starts with pristine 200K token context.

## When to Use
- **New implementation:** After planning, in fresh session (RECOMMENDED)
- **Retry:** After Critic or Testing feedback, to fix issues
- **Large implementations:** When context compactions are expected

---

## Execution Instructions

When the user says `/implement-phase`, references a plan file, or `/implement-phase #123`, execute these steps:

### Step 1: Determine Scenario

```
ANALYZE user message:

SCENARIO A: New Implementation - Plan File Referenced (BEST)
- User references file: "/implement-phase - use .claude/plan-X.md"
- No issue number
- BEST: Fresh session, complete plan from file

SCENARIO B: New Implementation - Plan Provided (RECOMMENDED)
- User provides plan in THIS message (multi-paragraph, structured)
- No issue number
- GOOD: Fresh session, no context bloat

SCENARIO C: New Implementation - Plan in Context (LEGACY)
- User says: "/implement-phase"
- No plan in current message
- No issue number
- WARNING: May be same session as planning (bloated)

SCENARIO D: Retry Existing (Has Issue Number)
- User says: "/implement-phase #123"
- Issue number provided
- Markdown file should already exist

IF issue number provided:
  GOTO: Step 2D (Retry Flow)
ELSE IF plan file referenced (e.g., ".claude/plan-X.md"):
  EXTRACT: File path → {plan_file}
  GOTO: Step 2A (File Reference Flow - BEST)
ELSE IF plan content in user's message:
  GOTO: Step 2B (Fresh Session Flow - RECOMMENDED)
ELSE:
  WARNING: "Context may be bloated. Consider referencing plan file."
  GOTO: Step 2C (Legacy Flow - Search Context)
```

---

## SCENARIO A: File Reference - Plan in File (BEST!)

### Step 2A: Read Plan from Referenced File

```
ACTION: User has referenced a plan file

EXTRACT: File path from user's message
- Examples: ".claude/plan-2026-01-14.md"
           "plan from .claude/plans/feature-X.md"
           "use plan-abc123.md"

READ FILE: {plan_file}

IF file not found:
  OUTPUT: "❌ Cannot find plan file: {plan_file}
  
  Please check:
  - File path is correct
  - File exists in the project
  
  Or paste the plan directly."
  STOP

IF file found:
  EXTRACT from file contents:
  - Requirements
  - Implementation approach
  - Acceptance criteria
  - Technical specifications
  
  STORE: {plan_content}
  
  This is the OPTIMAL scenario:
  ✅ Fresh session (pristine 200K tokens)
  ✅ Complete plan from file (no missing parts)
  ✅ No copy/paste errors
  ✅ Can reference later

PROCEED to Step 3A (Create GitHub Issue)
```

### Step 3A: Create GitHub Issue from Plan

```
ACTION: Create new GitHub issue in {github_repo}

EXTRACT TITLE:
- Look for plan title in first header or opening lines
- Generate concise title if not found (max 80 chars)
- Format: "Add [feature]" or "Implement [feature]"

GITHUB ISSUE STRUCTURE:
Repository: {github_repo}
Title: {extracted or generated title}
Body: {plan_content - full requirements and approach}
Labels: ["implementation"] (optional)

TOOL: Use GitHub MCP or API
EXECUTE: Create issue in {github_repo}

RESULT: Receive issue number → {issue_number}

ERROR HANDLING:
IF GitHub creation fails:
  OUTPUT: "⚠️ Could not create GitHub issue in {github_repo}
  
  Error: {error_message}
  
  Possible reasons:
  - GitHub MCP not configured
  - No access to {github_repo} repository
  - Network/authentication issue
  
  Options:
  1. Continue with local-only tracking (issue will be 'local-{timestamp}')
  2. Fix GitHub access and retry
  
  Continue without GitHub? (yes/no)"
  
  WAIT FOR USER RESPONSE:
    IF "yes":
      {issue_number} = "local-{timestamp}"
      {github_enabled} = false
      OUTPUT: "⚠️ Working in local-only mode. Issue: local-{timestamp}"
      PROCEED to Step 4A
    ELSE:
      OUTPUT: "Please fix GitHub access and try again."
      STOP

IF successful:
  {github_enabled} = true
  {issue_url} = "https://github.com/{github_repo}/issues/{issue_number}"
  OUTPUT: "✅ Created GitHub issue #{issue_number} in {github_repo}"
  PROCEED to Step 4A
```

### Step 4A: Create Local Markdown File

```
ACTION: Create directory if needed
EXECUTE: mkdir -p .claude/implementations/

ACTION: Create markdown file
FILE: .claude/implementations/issue-{issue_number}.md

CONTENT:
---
# Issue #{issue_number}: {title}

**Status:** Implementation in progress
**Created:** {ISO timestamp}
**GitHub:** {if github_enabled: issue_url, else: "Local only"}
**Repository:** {github_repo}

## Original Requirements

{plan_content}

---
*Phase Mode workflow begins below. Agents will append their outputs.*
---

SAVE FILE

OUTPUT: "✅ Created implementation file: .claude/implementations/issue-{issue_number}.md"
```

### Step 5A: Execute Implementation Agent

```
ACTION: Read and execute Implementation Agent in Phase Mode

FILE: .claude/agents/01-implementation-agent.md

CONTEXT:
"You are in PHASE MODE - New Implementation (Attempt 1).

Issue number: {issue_number}
Repository: {github_repo}
Markdown file: .claude/implementations/issue-{issue_number}.md

This is a fresh session. Follow Phase Mode instructions.
This is the FIRST attempt. Implement based on requirements in the markdown.

Begin now."
```

### Step 6A: Report Completion

```
OUTPUT:
"✅ Implementation Phase Complete - New Implementation

Issue: #{issue_number} created in {github_repo}
{if github_enabled: "GitHub: {issue_url}"}
Status: {status from Implementation Agent}
Files changed: {count}

Results written to: .claude/implementations/issue-{issue_number}.md

Next Steps:
{IF status is ✅ SUCCESS or ⚠️ PARTIAL:}
  Close this session and run: /critic-phase #{issue_number}

{IF status is ❌ BLOCKED:}
  Review blocker in markdown file.
  Resolve issue, then retry: /implement-phase #{issue_number}
"
```

---

## SCENARIO B: Fresh Session - Plan Provided (RECOMMENDED)

### Step 2B: Extract Plan from Current Message

```
ACTION: User has provided plan directly in their message

EXTRACT from current message:
- Requirements
- Implementation approach
- Acceptance criteria
- Technical specifications

STORE: {plan_content}

This is the OPTIMAL scenario:
✅ Fresh session (pristine 200K tokens)
✅ Plan explicitly provided (no searching)
✅ No context bloat from planning session

PROCEED to Step 3B (Create GitHub Issue)
```

### Step 3B: Create GitHub Issue from Provided Plan

```
ACTION: Create new GitHub issue in {github_repo}

EXTRACT TITLE:
- Look for plan title in first header or opening lines
- Generate concise title if not found (max 80 chars)
- Format: "Add [feature]" or "Implement [feature]"

GITHUB ISSUE STRUCTURE:
Repository: {github_repo}
Title: {extracted or generated title}
Body: {plan_content - full requirements and approach}
Labels: ["implementation"] (optional)

TOOL: Use GitHub MCP or API
EXECUTE: Create issue in {github_repo}

RESULT: Receive issue number → {issue_number}

ERROR HANDLING:
IF GitHub creation fails:
  OUTPUT: "⚠️ Could not create GitHub issue in {github_repo}
  
  Error: {error_message}
  
  Options:
  1. Continue with local-only tracking (issue will be 'local-{timestamp}')
  2. Fix GitHub access and retry
  
  Continue without GitHub? (yes/no)"
  
  WAIT FOR USER RESPONSE:
    IF "yes":
      {issue_number} = "local-{timestamp}"
      {github_enabled} = false
      OUTPUT: "⚠️ Working in local-only mode. Issue: local-{timestamp}"
      PROCEED to Step 4B
    ELSE:
      OUTPUT: "Please fix GitHub access and try again."
      STOP

IF successful:
  {github_enabled} = true
  {issue_url} = "https://github.com/{github_repo}/issues/{issue_number}"
  OUTPUT: "✅ Created GitHub issue #{issue_number} in {github_repo}"
  PROCEED to Step 4B
```

### Step 4B: Create Local Markdown File

```
ACTION: Create directory if needed
EXECUTE: mkdir -p .claude/implementations/

ACTION: Create markdown file
FILE: .claude/implementations/issue-{issue_number}.md

CONTENT:
---
# Issue #{issue_number}: {title}

**Status:** Implementation in progress
**Created:** {ISO timestamp}
**GitHub:** {if github_enabled: issue_url, else: "Local only"}
**Repository:** {github_repo}

## Original Requirements

{plan_content}

---
*Phase Mode workflow begins below. Agents will append their outputs.*
---

SAVE FILE

OUTPUT: "✅ Created implementation file: .claude/implementations/issue-{issue_number}.md"
```

### Step 5B: Execute Implementation Agent

```
ACTION: Read and execute Implementation Agent in Phase Mode

FILE: .claude/agents/01-implementation-agent.md

CONTEXT:
"You are in PHASE MODE - New Implementation (Attempt 1).

Issue number: {issue_number}
Repository: {github_repo}
Markdown file: .claude/implementations/issue-{issue_number}.md

This is a fresh session. Follow Phase Mode instructions.
This is the FIRST attempt. Implement based on requirements in the markdown.

Begin now."
```

### Step 6B: Report Completion

```
OUTPUT:
"✅ Implementation Phase Complete - New Implementation

Issue: #{issue_number} created in {github_repo}
{if github_enabled: "GitHub: {issue_url}"}
Status: {status from Implementation Agent}
Files changed: {count}

Results written to: .claude/implementations/issue-{issue_number}.md

Next Steps:
{IF status is ✅ SUCCESS or ⚠️ PARTIAL:}
  Close this session and run: /critic-phase #{issue_number}

{IF status is ❌ BLOCKED:}
  Review blocker in markdown file.
  Resolve issue, then retry: /implement-phase #{issue_number}
"
```

---

## SCENARIO C: Legacy Flow - Plan in Context

### Step 2C: Extract Plan from Context

```
ACTION: Search recent messages for implementation plan

LOOK FOR:
- "Implementation Plan", "Requirements", "Acceptance Criteria"
- Structured planning content
- User approval of plan

IF PLAN FOUND:
  EXTRACT:
    - Requirements
    - Implementation approach
    - Acceptance criteria
    - Technical specifications
  STORE: {plan_content}
  PROCEED to Step 3C

IF NO PLAN FOUND:
  OUTPUT: "❌ No implementation plan found in context.
  
  Options:
  1. Share the plan you'd like to implement
  2. If you have an existing issue, use: /implement-phase #123
  3. Run planning first
  
  Phase Mode requires a plan to start."
  STOP
```

### Step 3C: Create GitHub Issue from Plan

```
ACTION: Create new GitHub issue in {github_repo}

EXTRACT TITLE:
- Look for plan title in context
- Generate concise title if not found (max 80 chars)

GITHUB ISSUE STRUCTURE:
Repository: {github_repo}
Title: {extracted or generated title}
Body: {plan_content}
Labels: ["implementation"] (optional)

TOOL: Use GitHub MCP or API
EXECUTE: Create issue in {github_repo}

RESULT: Receive issue number → {issue_number}

ERROR HANDLING:
IF GitHub creation fails:
  OUTPUT: "⚠️ Could not create GitHub issue in {github_repo}
  
  Options:
  1. Continue with local-only tracking
  2. Fix GitHub access and retry
  
  Continue without GitHub? (yes/no)"
  
  WAIT FOR USER RESPONSE:
    IF "yes":
      {issue_number} = "local-{timestamp}"
      {github_enabled} = false
      PROCEED to Step 4C
    ELSE:
      STOP

IF successful:
  {github_enabled} = true
  {issue_url} = "https://github.com/{github_repo}/issues/{issue_number}"
  PROCEED to Step 4C
```

### Step 4C: Create Local Markdown File

```
ACTION: Create directory if needed
EXECUTE: mkdir -p .claude/implementations/

ACTION: Create markdown file
FILE: .claude/implementations/issue-{issue_number}.md

CONTENT:
---
# Issue #{issue_number}: {title}

**Status:** Implementation in progress
**Created:** {ISO timestamp}
**GitHub:** {if github_enabled: issue_url, else: "Local only"}
**Repository:** {github_repo}

## Original Requirements

{plan_content}

---
*Phase Mode workflow begins below. Agents will append their outputs.*
---

SAVE FILE
```

### Step 5C: Execute Implementation Agent

```
FILE: .claude/agents/01-implementation-agent.md

CONTEXT:
"You are in PHASE MODE - New Implementation (Attempt 1).

Issue number: {issue_number}
Repository: {github_repo}
Markdown file: .claude/implementations/issue-{issue_number}.md

This is a fresh session. Follow Phase Mode instructions.
This is the FIRST attempt. Implement based on requirements in the markdown.

Begin now."
```

### Step 6C: Report Completion

```
OUTPUT:
"✅ Implementation Phase Complete - New Implementation

Issue: #{issue_number} created in {github_repo}
{if github_enabled: "GitHub: {issue_url}"}
Status: {status from Implementation Agent}
Files changed: {count}

Results written to: .claude/implementations/issue-{issue_number}.md

Next Steps:
{IF status is ✅ SUCCESS or ⚠️ PARTIAL:}
  Close this session and run: /critic-phase #{issue_number}

{IF status is ❌ BLOCKED:}
  Review blocker in markdown file.
  Resolve issue, then retry: /implement-phase #{issue_number}
"
```

---

## SCENARIO D: Retry Existing Implementation

### Step 2D: Parse Issue Number

```
EXTRACT: Issue number from user message
- Format: "/implement-phase #123" or "/implement-phase 123"
- Store as: {issue_number}
```

### Step 3D: Verify Markdown File Exists

```
ACTION: Check if implementation file exists

FILE: .claude/implementations/issue-{issue_number}.md

IF NOT EXISTS:
  OUTPUT: "❌ Error: Cannot find .claude/implementations/issue-{issue_number}.md
  
  This file should exist for retries.
  
  Options:
  1. Check issue number is correct
  2. If this is NEW implementation, use: /implement-phase (without issue number)
  3. If file was deleted, you may need to reinitialize
  
  Note: /implement-phase without a number creates NEW implementations.
        /implement-phase #123 is for RETRYING existing implementations."
  STOP

IF EXISTS:
  PROCEED to Step 4D
```

### Step 4D: Execute Implementation Agent (Retry)

```
ACTION: Read and execute Implementation Agent in Phase Mode

FILE: .claude/agents/01-implementation-agent.md

CONTEXT:
"You are in PHASE MODE - Retry Implementation.

Issue number: {issue_number}
Repository: {github_repo}
Markdown file: .claude/implementations/issue-{issue_number}.md

This is a fresh session. Follow Phase Mode instructions.

This is a RETRY. Load context from markdown file:
- See previous implementation attempts
- See Critic feedback (if exists)
- See Testing results (if exists)
- Address the specific issues flagged

Determine attempt number and implement fixes.

Begin now."
```

### Step 5D: Report Completion

```
OUTPUT:
"✅ Implementation Phase Complete - Retry Attempt {N}

Issue: #{issue_number} in {github_repo}
Status: {status from Implementation Agent}
Files changed: {count}

Results written to: .claude/implementations/issue-{issue_number}.md

Next Steps:
{IF status is ✅ SUCCESS or ⚠️ PARTIAL:}
  Close this session and run: /critic-phase #{issue_number}

{IF status is ❌ BLOCKED:}
  Review blocker in markdown file.
  Resolve issue, then retry: /implement-phase #{issue_number}
"
```

---

## Error Handling

### No Plan and No Issue Number
```
"❌ Cannot proceed without either:
- A plan in context (for new implementation)
- An issue number (for retry)

What would you like to do?"
```

### Plan Exists But GitHub Creation Fails
```
"⚠️ GitHub issue creation failed but I have your plan.

Options:
1. Continue local-only (issue will be 'local-{timestamp}')
2. Fix GitHub and retry

Local-only mode works but won't sync with GitHub."
```

### Wrong Issue Number
```
"❌ Issue #999 markdown file not found.

Check:
- Is issue number correct?
- Was this issue initialized?

For NEW implementations: /implement-phase (no number)
For RETRIES: /implement-phase #999 (with number)"
```

---

**This command is now smart enough to handle both new implementations AND retries in Phase Mode, with full GitHub integration for {github_repo}!**