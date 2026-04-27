# Implement Command

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

**IMPORTANT: This command works best in a FRESH session after planning.**

If planning was lengthy, user has THREE options:

### Option 1: Reference Plan File (EASIEST ✅)
```
User: "/implement - use plan from .claude/plan-2026-01-14.md"
Command: Reads file, extracts plan, proceeds
```

### Option 2: Paste Plan in Message
```
User: [Pastes full plan text]
User: "/implement"
Command: Extracts from message, proceeds
```

### Option 3: Legacy (Same Session)
```
User: "/implement"
Command: Searches context (may be bloated)
```

**Recommended workflow:**
1. Complete planning in Session 1
2. Note the plan file path (if saved)
3. Close VS Code (end planning session)
4. Open fresh VS Code (Session 2)
5. Reference file OR paste plan
6. Run command

This ensures implementation starts with pristine context (no bloat from planning).

---

When the user says "Implement this plan", "/implement", "Implement issue #123", or references a plan file, execute these steps:

### Step 1: Detect Workflow Type

Analyze the user's message to determine which scenario applies:

**Scenario A: Plan file referenced (BEST)**
- User says: "/implement - use plan from .claude/plan-X.md"
- OR: "Implement using .claude/plan-X.md"
- Indicates: Fresh session, plan in file
- BEST: No context bloat, complete plan guaranteed

**Scenario B: Plan provided in THIS message (RECOMMENDED)**
- User shares plan in their message (pasted from planning session)
- Indicates: Fresh session, plan explicitly provided
- GOOD: No context bloat from planning

**Scenario C: Plan in recent context (LEGACY - may have bloat)**
- User says: "Implement this plan", "Let's implement"
- No plan in current message, no file reference
- Indicates: Same session as planning (may be bloated)

**Scenario D: Resuming existing work (issue number given)**
- User says: "Implement issue #123", "Continue issue #456"
- Indicates: Fetch from existing GitHub issue

```
IF user message contains issue number pattern (e.g., "#123", "issue 123"):
  scenario = "RESUME_EXISTING"
  EXTRACT: Issue number → {issue_number}
  GOTO: Step 2D (Resume Flow)
ELSE IF user message references file (e.g., ".claude/plan-X.md", "plan from file"):
  scenario = "NEW_FROM_FILE"
  EXTRACT: File path → {plan_file}
  GOTO: Step 2A (File Reference Flow)
ELSE IF user message contains plan content (multi-paragraph, structured):
  scenario = "NEW_FROM_PROVIDED_PLAN"
  GOTO: Step 2B (Fresh Session Flow)
ELSE:
  scenario = "NEW_FROM_CONTEXT_PLAN"
  WARNING: "Context may be bloated from planning session"
  GOTO: Step 2C (Legacy Flow)
```

---

## PATH A: File Reference - Plan in File (BEST!)

### Step 2A: Read Plan from Referenced File

```
ACTION: User has referenced a plan file

EXTRACT: File path from user's message
- Examples: ".claude/plan-2026-01-14.md"
           ".claude/plans/feature-X.md"
           "plan-abc123.md"

READ FILE: {plan_file}

IF file not found:
  OUTPUT: "❌ Cannot find plan file: {plan_file}
  
  Please check:
  - File path is correct
  - File exists in the project
  
  Or paste the plan directly in your message."
  STOP

IF file found:
  EXTRACT from file contents:
  - Implementation requirements
  - Technical approach
  - Acceptance criteria
  - Any architectural decisions
  
  STORE: {plan_content}
  
  This is the BEST scenario:
  ✅ Fresh session (no context bloat)
  ✅ Complete plan from file
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
      PROCEED to Step 3 (Common Path)
    ELSE:
      OUTPUT: "Please fix GitHub access and try again."
      STOP

IF successful:
  {github_enabled} = true
  {issue_url} = "https://github.com/{github_repo}/issues/{issue_number}"
  OUTPUT: "✅ Created GitHub issue #{issue_number} in {github_repo}"
  PROCEED to Step 3 (Common Path)
```

---

## PATH B: Fresh Session - Plan Provided in Message (GOOD)

### Step 2B: Extract Plan from Current Message

```
ACTION: User has provided plan directly in their message

EXTRACT from current message:
- Implementation requirements
- Technical approach
- Acceptance criteria
- Any architectural decisions

STORE: {plan_content}

This is the BEST scenario:
✅ Fresh session (no context bloat)
✅ Plan explicitly provided
✅ Implementation starts with pristine 200K tokens

PROCEED to Step 3A (Create GitHub Issue)
```

### Step 3A: Create GitHub Issue from Provided Plan

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
      PROCEED to Step 3 (Common Path)
    ELSE:
      OUTPUT: "Please fix GitHub access and try again."
      STOP

IF successful:
  {github_enabled} = true
  {issue_url} = "https://github.com/{github_repo}/issues/{issue_number}"
  OUTPUT: "✅ Created GitHub issue #{issue_number} in {github_repo}"
  PROCEED to Step 3 (Common Path)
```

---

## PATH B: Legacy Flow - Plan in Context (May Have Bloat)

### Step 2B: Detect and Extract Plan from Context

```
ACTION: Search recent conversation messages for implementation plan

LOOK FOR:
- Messages containing "Implementation Plan", "Acceptance Criteria", "Requirements"
- Structured content with goals, technical approach, tasks
- Lists of acceptance criteria
- Technical specifications
- Usually from Claude Code's plan mode output

PLAN DETECTION PATTERNS:
- Headers like "## Implementation Plan", "## Technical Approach", "## Requirements"
- Bulleted or numbered lists of tasks
- Acceptance criteria sections
- User confirmation like "looks good", "approved", "let's do this"

EXTRACT:
- Full plan content
- Requirements and acceptance criteria
- Technical approach
- Any architectural decisions

STORE as: {plan_content}

IF plan found:
  PROCEED to Step 2A.2
ELSE:
  OUTPUT: "I don't see a plan in our recent conversation.
  
  Options:
  1. Share the plan now and I'll implement it
  2. Provide a GitHub issue number: 'Implement issue #123'
  3. Run planning first using Claude Code's plan mode
  
  Which would you like to do?"
  
  WAIT FOR USER RESPONSE:
  - If plan shared: Extract and continue to Step 2A.2
  - If issue number given: Switch to Step 2B
  - If neither: STOP
```

### Step 2A.2: Create GitHub Issue from Plan

```
ACTION: Create new GitHub issue using the plan from context

ISSUE STRUCTURE:
Repository: {github_repo}
Title: {extract from plan or generate meaningful title}
Body: {plan_content} - Full plan from context
Labels: ["implementation", "planned"] (optional)

TOOL: GitHub MCP or API
EXECUTE: Create issue in {github_repo}

RESPONSE: Receive issue number → {issue_number}

ERROR HANDLING:
IF GitHub issue creation fails:
  OUTPUT: "⚠️ Could not create GitHub issue: {error_message}
  
  This might be because:
  - GitHub MCP not configured
  - No repository access
  - Network issue
  
  Options:
  1. Continue with local-only tracking (no GitHub issue)
  2. Fix GitHub setup and try again
  3. Manually create issue and provide number
  
  Continue without GitHub? (yes/no)"
  
  WAIT FOR USER:
    IF "yes":
      {issue_number} = "local-{timestamp}"
      {github_enabled} = false
      OUTPUT: "⚠️ Proceeding without GitHub tracking. Issue will be local-only."
      PROCEED to Step 3
    ELSE:
      STOP with message: "Fix GitHub setup and run 'Implement' again"

IF successful:
  {github_enabled} = true
  OUTPUT: "✅ Created GitHub issue #{issue_number}: {title}"
  PROCEED to Step 3
```

---

## PATH B: Resume from Existing GitHub Issue

### Step 2B: Fetch Existing GitHub Issue

```
PREREQUISITE: {issue_number} already extracted from user message

ACTION: Fetch existing GitHub issue from {github_repo}

TOOL: GitHub MCP or API
EXECUTE: Fetch issue #{issue_number} from {github_repo}

RETRIEVE:
- Issue title
- Issue body (should contain plan/requirements)
- Current status
- Labels
- Any existing comments with context

STORE: {plan_content} = issue body content

ERROR HANDLING:
IF fetch fails:
  OUTPUT: "❌ Cannot fetch issue #{issue_number}
  
  Possible reasons:
  - Issue doesn't exist
  - No repository access
  - GitHub MCP not configured
  - Wrong issue number
  
  Please verify:
  1. Issue number is correct: #{issue_number}
  2. You have access to the repository
  3. GitHub integration is configured
  
  Would you like to try a different issue number?"
  STOP

IF successful:
  {github_enabled} = true
  OUTPUT: "✅ Fetched issue #{issue_number}: {title}"
  PROCEED to Step 3
```

---

## COMMON PATH: Both flows converge here

### Step 3: Check for Existing Workflow

```
ACTION: Check if markdown file already exists

FILE: .claude/implementations/issue-{issue_number}.md

IF file exists:
  OUTPUT: "⚠️ Implementation already in progress for issue #{issue_number}
  
  Found existing file: .claude/implementations/issue-{issue_number}.md
  
  Options:
  1. Resume existing workflow (keep existing file)
  2. Restart fresh (archive old file and start over)
  
  Reply: resume / restart"
  
  WAIT FOR USER:
    IF "resume":
      OUTPUT: "Resuming existing workflow. Loading Orchestrator..."
      GOTO: Step 5 (skip markdown creation, go straight to Orchestrator)
    
    IF "restart":
      TIMESTAMP = current ISO timestamp
      OLD_FILE = .claude/implementations/issue-{issue_number}-archived-{TIMESTAMP}.md
      RENAME: existing file → OLD_FILE
      OUTPUT: "✅ Archived old implementation to: {OLD_FILE}"
      PROCEED to Step 4
    
    ELSE:
      Re-prompt user to choose resume or restart

IF file does not exist:
  PROCEED to Step 4
```

### Step 4: Create Local Markdown File

```
ACTION: Create directory if needed
EXECUTE: mkdir -p .claude/implementations/

ACTION: Create markdown file
FILE: .claude/implementations/issue-{issue_number}.md

CONTENT STRUCTURE:
---
# Issue #{issue_number}: {title}

**Status:** Implementation in progress
**Created:** {timestamp}
**GitHub:** {if github_enabled: issue URL, else: "Local only"}
**Repository:** {github_repo}

## Original Requirements

{plan_content}

---
*Implementation workflow begins below. Agents will append their outputs.*
---

SAVE FILE
```

### Step 5: Confirm Initialization

```
OUTPUT FORMAT:
"━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ IMPLEMENTATION INITIALIZED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Issue: #{issue_number} - {title}
{if github_enabled: "GitHub: " + issue_url}
Markdown: .claude/implementations/issue-{issue_number}.md

Plan Source: {if PATH A: "✅ Loaded from context (no fetch needed)" else: "✅ Fetched from GitHub issue"}

The Orchestrator will now manage:
├─ Implementation Agent → Write code
├─ Critic Agent → Review quality  
├─ Testing Agent → Validate & test
└─ Checkpoints → Pause for your review

Starting implementation workflow...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

### Step 6: Load and Execute Orchestrator Agent

```
ACTION: Read orchestrator agent instructions
FILE: Read 00-orchestrator-agent.md

CONTEXT TO PASS TO ORCHESTRATOR:
- Issue number: {issue_number}
- Markdown file path: .claude/implementations/issue-{issue_number}.md
- GitHub enabled: {github_enabled}
- Command: "Execute full implementation workflow"
- Starting stage: "Implementation Agent"

INSTRUCTION TO ORCHESTRATOR:
"You are the Orchestrator Agent. Execute the implementation workflow for issue #{issue_number} according to your instructions in 00-orchestrator-agent.md.

The markdown file is ready at: .claude/implementations/issue-{issue_number}.md

Begin by invoking the Implementation Agent to write code based on the requirements in the markdown file."

HANDOFF: Control passes to Orchestrator Agent
YOUR ROLE (Implement Command): Complete - Orchestrator takes over
```

---

## Summary of What This Command Does

### For "Implement this plan" (Just finished planning):
1. ✅ Extracts plan from recent context (NO GitHub fetch)
2. ✅ Creates GitHub issue from the plan
3. ✅ Creates local markdown from context plan
4. ✅ Hands off to Orchestrator

**Benefits:**
- No manual copy/paste to GitHub
- No redundant API fetch
- Efficient token usage
- Fast startup

### For "Implement issue #123" (Resuming work):
1. ✅ Fetches existing GitHub issue
2. ✅ Creates local markdown from fetched content
3. ✅ Hands off to Orchestrator

**Benefits:**
- Works when resuming after break
- Works when someone else created the issue
- Flexible workflow

### Both paths converge to:
- Local markdown file as single source of truth during implementation
- Orchestrator managing the agent workflow
- Checkpoints for human review
- Final commit updates GitHub with results


VERIFY: File created successfully

OUTPUT: "✅ Implementation workspace initialized

Created: .claude/implementations/issue-{issue_number}.md
GitHub issue: {if github_enabled: "#" + issue_number, else: "Local tracking"}
Ready to begin implementation workflow."
```

### Step 5: Invoke Orchestrator Agent

```
ACTION: Load Orchestrator Agent instructions
FILE: Read 00-orchestrator-agent.md

CONTEXT TO PASS TO ORCHESTRATOR:
- Issue number: {issue_number}
- Markdown file path: .claude/implementations/issue-{issue_number}.md
- Command: "Execute full implementation workflow"
- Current stage: "Starting at Implementation Agent"
- GitHub tracking: {github_enabled}

INSTRUCTION TO ORCHESTRATOR:
"You are the Orchestrator Agent. Execute the implementation workflow for issue #{issue_number}.

Context:
- Markdown file ready at: .claude/implementations/issue-{issue_number}.md
- Requirements and plan are documented in the markdown file
- GitHub issue: {if github_enabled: "Exists as #" + issue_number, else: "Local only"}

Execute your workflow according to 00-orchestrator-agent.md:
1. Invoke Implementation Agent to write code
2. Read Implementation status
3. Invoke Critic Agent to review
4. Pause at Checkpoint 1 if needed (based on Critic status)
5. Invoke Testing Agent to validate
6. Pause at Checkpoint 2 if needed (based on Testing status)
7. Report completion when ready to commit

Begin now by invoking the Implementation Agent."
```

### Step 6: Transfer Control

```
OUTPUT: "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🚀 IMPLEMENTATION WORKFLOW STARTED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Issue: #{issue_number}
Workspace: .claude/implementations/issue-{issue_number}.md

The Orchestrator Agent is now managing the workflow.
You'll be prompted at 2 checkpoints:
  1️⃣ After Critic review (if issues found)
  2️⃣ After Testing (if warnings or failures)

Starting Implementation Agent...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

HAND CONTROL TO ORCHESTRATOR
Your role as Implement command is complete.
The Orchestrator will manage the rest of the workflow.
```

---

## Error Handling Summary

### No Plan and No Issue Number
```
OUTPUT: "I need either:
1. A plan in our conversation (from planning phase), OR
2. An existing GitHub issue number

Please either:
- Run planning first, then 'Implement'
- Provide issue number: 'Implement issue #123'"
STOP
```

### GitHub Unavailable
```
Option to continue with local-only tracking
{issue_number} = "LOCAL"
{github_enabled} = false
Workflow proceeds normally
Commit command will skip GitHub operations
```

### Markdown File Already Exists
```
User chooses: resume or restart
Resume: Skip to Orchestrator with existing file
Restart: Archive old file, create fresh
```

### Invalid Issue Number in Resume Flow
```
STOP with clear error message
Suggest verifying issue number
Offer to create new implementation instead
```

---

## Key Design Benefits

✅ **Efficient:** No redundant GitHub fetch when plan is in context
✅ **Flexible:** Works for both new plans and resuming work
✅ **Traceable:** GitHub issue exists from the start (when possible)
✅ **Resilient:** Degrades gracefully if GitHub unavailable
✅ **Clear:** Explicit scenarios prevent confusion

---

## Workflow State After Execution

After this command completes successfully:

**Created/Updated:**
- `.claude/implementations/issue-{issue_number}.md` exists with plan
- GitHub issue #{issue_number} exists (if GitHub enabled)

**Control transferred to:**
- Orchestrator Agent (managing Implementation → Critic → Testing flow)

**User interaction points:**
- Checkpoint 1: After Critic (if issues found)
- Checkpoint 2: After Testing (if warnings/failures)

**Next command:**
- `Commit issue #{issue_number}` (after Orchestrator reports completion)