# Commit Phase Command

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

Runs the Commit command in a fresh session. Performs final validation, documents in GitHub, commits code, closes issue.

## When to Use

- After `/test-phase` reports ✅ SUCCESS
- As final phase in manual multi-session workflow
- When ready to finalize implementation

---

## Execution Instructions

### Step 1: Parse Issue Number

```
EXTRACT: {issue_number} from command
Format: /commit-phase #123
```

### Step 2: Execute Commit Command in Phase Mode

```
FILE: .claude/commands/06-commit-command.md

INSTRUCTION: "Execute commit workflow for issue #{issue_number}.
Load context from .claude/implementations/issue-{issue_number}.md
This is a fresh session - treat as standalone commit operation."
```

---

## What Commit Command Will Do

The existing 06-commit-command.md will execute:

### Phase 1: Final Validation (Read-Only)

1. Verify Testing status is ✅ SUCCESS
2. Run TypeScript type check (if applicable)
3. Verify acceptance criteria met
4. Check for uncommitted changes

### Phase 2: Pre-Commit Review (Human Checkpoint)

Shows summary:

```
"Ready to Commit - Final Review

Issue: #{issue_number}
Files changed: {count}
Tests: All passing
Acceptance criteria: All met

Proceed? (yes/no)"
```

You must confirm "yes"

### Phase 3: Documentation Check

Checks if documentation needs updating across all three docs:

- `CLAUDE.md` — coding conventions, dev workflow
- `AGENTS.md` — GPT hybrid workflow guide
- `docs/context/{ProjectName}.md` — project manifest

Also checks CLAUDE.md health (bloat thresholds: 🟢 <800 / 🟡 800-1200 / 🔴 >1200 lines).

### Phase 4: Git Commit

```
git add [changed files]
git commit -m "[#{issue_number}] {title}"
```

### Phase 5: Close GitHub Issue

- Documents everything in GitHub
- Closes the issue
- Reports commit SHA

### Phase 6: Documentation Check

ACTION: Evaluate if this implementation requires documentation updates

The project has three documentation files with distinct scopes:

| Doc            | Path          | Scope                                                    |
| -------------- | ------------- | -------------------------------------------------------- |
| CLAUDE.md      | project root  | Coding conventions, dev workflow, slash commands         |
| AGENTS.md      | project root  | GPT hybrid workflow guide (architect & critic)           |
| {ProjectName}.md | docs/context/ | Project manifest: tech stack, domain logic, architecture |

Additionally, constraint docs live in .claude/docs/0X-\*.md.

## **Phase 6.1: Analyze Implementation Impact**

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
PROCEED to Phase 6.2

IF NO to all questions:
This is a minor change (bug fix, UI tweak, small refactor)
SKIP documentation check
PROCEED to Phase 7

---

## **Phase 6.2: Prompt User for Documentation**

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
PROCEED to Phase 6.3

IF "skip" or "no" or "2":
LOG: "User chose to skip documentation"
PROCEED to Phase 7

---

## **Phase 6.3: Draft and Apply Documentation Updates**

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

### Phase 7: Archive Implementation File

```
ACTION: Archive implementation file (MOVE, do NOT copy)
  IMPORTANT: Use `git mv` to move the file so git tracks the rename in a single operation.
  Do NOT use `cp` — the source file must be removed from implementations/.

  EXECUTE:
  mkdir -p .claude/implementations/archive/
  git mv .claude/implementations/issue-{issue_number}.md \
     .claude/implementations/archive/issue-{issue_number}.md
  git commit -m "chore: archive issue-{issue_number} implementation file"
```

### Phase 8: Report Completion

```
"COMMIT COMPLETE

Commit: {sha}
Issue: #{issue_number} closed
Archive: .claude/implementations/archive/issue-{issue_number}.md
{if docs updated: "Documentation: Updated ({list updated docs})"}
{else: "Documentation: No updates needed"}

Next: git push origin {branch}"
```

---

## Usage Example

```
[After testing passed]

User in fresh session: "/commit-phase #123"

[Validation runs]
[Shows summary]
Commit: "Proceed? (yes/no)"

User: "yes"

[Checks CLAUDE.md]
Commit: "No significant changes detected. No update needed."
[Commits code]
[Closes GitHub issue]
Commit: "✅ COMMIT COMPLETE - Commit: abc123"

User: git push origin main
Done!
```

---

## CLAUDE.md Auto-Update Example

```
User: "/commit-phase #456"

[Validation passes]
[Shows summary]
User: "yes"

[Checks CLAUDE.md]
Commit: "📝 CLAUDE.md UPDATE CHECK

Current: 650 lines [🟢 Healthy]

Detected changes:
- New dependency: three.js (r128)
- New file structure: src/3d/

Options:
1. Update CLAUDE.md (+2 lines)
2. Skip

Reply: 1"

User: "1"

[Updates CLAUDE.md]
[Commits both code + CLAUDE.md]
[Closes issue]

Done!
```

---

## Error Handling

**Testing Not Complete:**

```
"❌ Cannot commit: Testing status is not ✅ SUCCESS

Run /test-phase #{issue_number} first and ensure it passes."
STOP
```

**Type Errors:**

```
"❌ TypeScript errors detected:
[list errors]

Fix errors and re-run /test-phase, then retry /commit-phase."
STOP
```

**No Changes:**

```
"❌ No changes to commit.
git status shows no modified files."
STOP
```

---

## Integration with Workflow

**Final phase:**

```
Session 1: /implement-phase #123 → Close
Session 2: /critic-phase #123 → Close
Session 3: /test-phase #123 → Close
Session 4: /commit-phase #123 → Close ← YOU ARE HERE

[Finalization complete]
User: git push
Done! 🎉
```

**Fresh context benefit:**

- Commit phase gets full 200K tokens
- Can handle large file lists
- No context degradation from previous phases
- Clear, focused on finalization

---

## Key Differences from Orchestrator Mode

| Feature             | Orchestrator Mode              | Phase Mode          |
| ------------------- | ------------------------------ | ------------------- |
| **Context**         | Accumulated from full workflow | Fresh from markdown |
| **Validation**      | May be affected by compaction  | Full token budget   |
| **CLAUDE.md check** | Same logic                     | Same logic          |
| **Final step**      | In same session                | In fresh session    |

---

## After This Command

**Workflow is complete!**

Next steps:

1. Review commit: `git show {sha}`
2. Push to remote: `git push origin {branch}`

**Note:** Implementation file automatically archived to `.claude/implementations/archive/issue-{number}.md`

**The multi-session workflow is done. Each phase had fresh context, no compactions occurred, high quality maintained throughout!**

---

**Remember:** This is the FINAL phase. After this completes, your implementation is committed, documented, and the GitHub issue is closed. All that's left is to push!
