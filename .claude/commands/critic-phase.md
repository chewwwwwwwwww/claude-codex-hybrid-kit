# Critic Phase Command

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
This command runs ONLY the Critic Agent in a fresh session as part of a manual multi-session workflow. Use this to review implementation with fresh context.

## When to Use
- After `/implement-phase` completes
- As part of manual session workflow for large implementations
- When you want fresh context for code review

---

## Execution Instructions

When the user says `/critic-phase #123`, execute these steps:

### Step 1: Parse Issue Number

```
EXTRACT: Issue number from command
- Format: `/critic-phase #123` or `/critic-phase 123`
- Store as: {issue_number}
```

### Step 2: Verify Markdown File and Implementation Exists

```
ACTION: Check if implementation file exists and has implementation notes

FILE: .claude/implementations/issue-{issue_number}.md

IF NOT EXISTS:
  OUTPUT: "❌ Error: Cannot find .claude/implementations/issue-{issue_number}.md
  
  Run /implement-phase #123 first."
  STOP

IF EXISTS:
  READ FILE
  CHECK: Does it have "## Implementation Notes" section?
  
  IF NO:
    OUTPUT: "❌ Error: No implementation found to review.
    
    The markdown exists but has no Implementation Notes section.
    
    Run /implement-phase #{issue_number} first."
    STOP
  
  IF YES:
    PROCEED to Step 3
```

### Step 3: Load and Execute Critic Agent

```
ACTION: Read and execute the Critic Agent in Phase Mode

FILE TO EXECUTE: .claude/agents/02-critic-agent.md

CONTEXT TO PROVIDE:
"You are being invoked in PHASE MODE.

Issue number: {issue_number}
Markdown file: .claude/implementations/issue-{issue_number}.md

This is a fresh session. Follow the PHASE MODE instructions in your file.

Load context from the markdown, review the implementation, write results back.

Begin now."
```

### Step 4: Critic Agent Executes

The Critic Agent will:
1. Read .claude/implementations/issue-{issue_number}.md
2. Load implementation notes and files changed
3. Review code for quality, correctness, architecture, security
4. Categorize issues (🔴 Critical, 🟡 Important, 🟢 Minor)
5. Write review to markdown file
6. Report completion with recommendation

### Step 5: Completion

After Critic Agent finishes, it will report status and guide next steps.

---

## Usage Examples

### Example 1: Review After Implementation

```
[Previous session: /implement-phase #123 completed]

User in fresh session: "/critic-phase #123"

[Critic Agent loads context]
[Reviews code]
[Finds 2 important issues]
[Documents review]
[Reports: "⚠️ PARTIAL - Found important issues. Recommend retry."]

User decides: retry implementation or proceed to testing
User closes VS Code
```

### Example 2: Review After Implementation Retry

```
[Implementation Attempt 2 after fixing Critic feedback]

User in fresh session: "/critic-phase #123"

[Critic Agent loads context]
[Sees this is review of Attempt 2]
[Verifies previous issues were fixed]
[Reports: "✅ SUCCESS - Issues resolved. Proceed to testing."]

User closes VS Code
```

### Example 3: Multiple Iterations

```
Session 1: /implement-phase #123 → Close
Session 2: /critic-phase #123 → "⚠️ PARTIAL" → Close
Session 3: /implement-phase #123 (retry) → Close
Session 4: /critic-phase #123 → "⚠️ PARTIAL still" → Close
Session 5: /implement-phase #123 (retry again) → Close
Session 6: /critic-phase #123 → "✅ SUCCESS" → Close
Session 7: /test-phase #123 → Continue...

Each review gets fresh context!
```

---

## Critic Agent Status Meanings

**✅ SUCCESS:**
- No critical or important issues
- Only minor suggestions (optional fixes)
- Ready to proceed to testing
- **Next step:** Close session, run `/test-phase #123`

**⚠️ PARTIAL:**
- Important issues found (but not critical)
- Core approach is sound but needs refinement
- **Decision needed:**
  - Proceed to testing anyway → `/test-phase #123`
  - Fix issues first → `/implement-phase #123` (retry)

**❌ BLOCKED:**
- Critical issues that prevent testing
- Fundamental problems requiring rethinking
- **Next step:** Close session, run `/implement-phase #123` (retry)

---

## Error Handling

### Error: No Implementation to Review
```
"❌ No implementation found in .claude/implementations/issue-{issue_number}.md

The file exists but has no Implementation Notes.

Run /implement-phase #{issue_number} first."
STOP
```

### Error: File Not Found
```
"❌ Cannot find .claude/implementations/issue-{issue_number}.md

Check the issue number or run /implement to initialize."
STOP
```

---

## Decision Making After Critic

**If Critic says ✅ SUCCESS:**
→ Proceed to `/test-phase #123`

**If Critic says ⚠️ PARTIAL:**
→ You decide:
- Issues are acceptable → `/test-phase #123`
- Issues need fixing → `/implement-phase #123` (creates Attempt N+1)

**If Critic says ❌ BLOCKED:**
→ Must fix critical issues → `/implement-phase #123` (retry)

---

## Integration with Manual Session Workflow

**Position in workflow:**
```
Session 1: /implement-phase #123 → Close
Session 2: /critic-phase #123 → Close ← YOU ARE HERE
Session 3: [Based on Critic result]
  IF ✅ or acceptable ⚠️: /test-phase #123
  IF needs fixing: /implement-phase #123 (retry, back to Session 1)
```

**Fresh context benefit:**
- Critic reviews with full 200K tokens
- No context degradation from implementation phase
- Better quality review
- Can handle reviewing very large implementations

---

## Key Differences from Orchestrator Mode

| Feature | Orchestrator Mode | Phase Mode |
|---------|-------------------|------------|
| **Context** | Already loaded | Must load from markdown |
| **Session** | Continuous | Fresh |
| **Next step** | Orchestrator decides | Human decides |
| **Retry** | Orchestrator manages | Human runs /implement-phase |

---

**Remember:** This is ONE phase in a multi-phase workflow. After Critic completes, you close VS Code and decide the next phase based on the results.