# Update CLAUDE.md Command

## Path Resolution (IMPORTANT)

This command operates on the **current project's** CLAUDE.md file, not a global one.
```
FILE PATHS:
  CLAUDE.md               →  {current_project}/CLAUDE.md
  CLAUDE-v{N}.md          →  {current_project}/CLAUDE-v{N}.md

EXAMPLE (working on any project):
  CLAUDE.md resolves to: {project_name}/CLAUDE.md
  NOT: ~/.claude/CLAUDE.md or ~/CLAUDE.md

Each project has its own CLAUDE.md in its root directory.
```

## GitHub Repository Resolution (CRITICAL)

**Before any GitHub operations, determine the correct repository:**

1. **Check CLAUDE.md** (in project root):
   - Look for line starting with `**Repository:**`
   - Example: `**Repository:** owner/repo-name`
   - This is the PREFERRED method - project explicitly declares its repo

2. **Fall back to git remote**:
   - Run: `git remote get-url origin`
   - Parse: `git@github.com:owner/repo.git` → `owner/repo`
   - Or: `https://github.com/owner/repo.git` → `owner/repo`

3. **Store as {github_repo}** for all GitHub operations

**NEVER hardcode a repository name. Always resolve dynamically.**

---

## Purpose
Manages CLAUDE.md updates and archival when the file grows too large. Prevents context bloat by archiving old sections and refreshing with current project state.

## When to Use
- After committing an implementation that modified project structure
- When CLAUDE.md grows beyond 1200 lines (critical bloat)
- When user explicitly asks to update/refresh CLAUDE.md
- Periodically to keep project documentation current

## Execution Instructions

### Step 1: Measure Current CLAUDE.md Size

```
ACTION: Read CLAUDE.md
FILE: CLAUDE.md (project root)

MEASURE BLOAT:
- Count total lines: {line_count}
- Check section sizes

BLOAT THRESHOLDS:
- 🟢 Healthy: <800 lines
- 🟡 Warning: 800-1200 lines  
- 🔴 Critical: >1200 lines

OUTPUT:
"CLAUDE.md Status:
- Size: {line_count} lines [{🟢/🟡/🔴} {status}]
- Last modified: {date}

{IF 🔴 Critical bloat:}
🔴 CRITICAL: CLAUDE.md has grown large (>{1200} lines)
Recommendation: Archive old version and refresh

{IF 🟡 Warning:}
🟡 WARNING: CLAUDE.md is growing (800-1200 lines)
Consider archival if making major additions
"
```

---

### Step 2: Determine Update Strategy

```
ANALYZE what needs updating:
- New features/functionality added
- Tech stack changes
- Workflow changes
- Slash command changes
- Agent additions/modifications
- Constraint document additions

PRESENT OPTIONS TO USER:

{IF 🟢 Healthy (<800 lines):}
"How would you like to proceed?

1. Update CLAUDE.md (add {N} lines, keeps concise)
2. Update + Archive (create version, refresh CLAUDE.md)

Recommendation: Option 1 (Simple update)"

{IF 🟡 Warning (800-1200 lines):}
"How would you like to proceed?

1. Update CLAUDE.md (add {N} lines, may push toward critical)
2. Update + Archive (create version, refresh CLAUDE.md)

{IF major_change: "Recommendation: Option 2 (Update + Archive)"}
{ELSE: "Recommendation: Option 1 (Update only)"}"

{IF 🔴 Critical (>1200 lines):}
"CLAUDE.md is critically bloated. You should archive.

1. Update CLAUDE.md (add {N} lines, stays bloated)
2. Update + Archive (create version, refresh CLAUDE.md)

{ELSE IF Critical bloat: "Recommendation: Option 2 (Update + Archive)"}
**Strongly Recommended:** Option 2 (Archive + Refresh)"

WAIT FOR USER DECISION
```

---

### Step 3A: Simple Update (Option 1)

```
ACTION: Update CLAUDE.md in place

GUIDELINES:
- Keep changes minimal and focused
- Update only changed sections
- Don't add unnecessary detail
- Maintain existing structure
- Keep each addition under 3 lines

SECTIONS TO UPDATE (if applicable):
1. Tech Stack (if dependencies changed)
2. Slash Commands (if new commands added)
3. Agent Files (if agents added/renamed)
4. Workflow Diagrams (if workflow changed)
5. Key Features (if major features added)

EXECUTION:
FOR EACH section needing update:
  READ: Current content
  IDENTIFY: What changed
  UPDATE: Minimal changes only
  VERIFY: Section remains concise

FINAL:
  COUNT: Total lines added
  VERIFY: Total file size
  OUTPUT: "✅ CLAUDE.md updated (+{lines_added} lines, now {new_total} lines)"
```

---

### Step 3B: Update + Archive (Option 2)

```
ACTION: Archive current CLAUDE.md and create fresh version

STEP 1: Determine version number
  CHECK: Existing archived versions
  PATTERN: CLAUDE-v{N}.md
  EXAMPLES: CLAUDE-v1.md, CLAUDE-v2.md
  NEXT: Increment to v{N+1}

STEP 2: Archive current
  FILE: CLAUDE.md
  RENAME TO: CLAUDE-v{N}.md
  LOCATION: Same directory as CLAUDE.md
  
  ADD HEADER to archived file:
    "# CLAUDE.md - Version {N} (Archived)
    **Archived on:** {date}
    **Reason:** File size exceeded {line_count} lines
    
    This is an archived version for historical reference.
    See CLAUDE.md for current documentation.
    
    ---
    [Original content below]
    "

STEP 3: Create fresh CLAUDE.md
  ACTION: Build new CLAUDE.md from scratch
  
  INCLUDE (keep concise):
  1. Project Overview (2-3 sentences)
  2. Tech Stack (current only, no historical)
  3. Key Directories (essential paths)
  4. Slash Commands (current workflow)
  5. Agent Files (current agents only)
  6. Critical Constraints (reference .claude/docs/, don't duplicate)
  7. Workflow Diagrams (current workflow)
  8. Key Features (top 5-7 only)
  
  EXCLUDE (move to archive):
  - Historical decisions
  - Old workflow descriptions
  - Deprecated commands
  - Outdated tech stack info
  - Verbose explanations (reference docs instead)
  - Implementation examples (reference WORKFLOW.md)
  
  TARGET SIZE: 400-600 lines (fresh start)

STEP 4: Verify new file
  CHECK: All essential sections present
  CHECK: Slash commands documented
  CHECK: File size <800 lines
  CHECK: References to detailed docs (.claude/WORKFLOW.md, etc.)

STEP 5: Update references
  IF other files reference CLAUDE.md sections:
    UPDATE: References to point to correct location
    (Most should reference WORKFLOW.md or other docs)

OUTPUT:
"✅ CLAUDE.md archived and refreshed

Archived: CLAUDE-v{N}.md ({old_line_count} lines)
New: CLAUDE.md ({new_line_count} lines)
Reduction: {reduction_percentage}%

The new CLAUDE.md is concise and current.
Historical context preserved in CLAUDE-v{N}.md."
```

---

## Best Practices for CLAUDE.md Maintenance

### Keep It Concise
- Reference other docs instead of duplicating
- Use `.claude/WORKFLOW.md` for detailed workflow
- Use `.claude/docs/` for constraints
- CLAUDE.md is **overview and entry point**, not comprehensive docs

### What Belongs in CLAUDE.md
✅ **Include:**
- High-level project purpose
- Current tech stack (just list)
- Slash commands (name + one line description)
- Agent file locations
- Directory structure (key paths only)
- How to get started

❌ **Exclude:**
- Detailed workflow instructions (→ WORKFLOW.md)
- Constraint explanations (→ .claude/docs/)
- Historical decisions (→ archives)
- Implementation examples (→ WORKFLOW.md)
- Verbose explanations

### Archival Triggers
Archive when:
- 🔴 File exceeds 1200 lines
- 🟡 File at 800-1200 lines AND making major additions (>100 lines)
- 🔴 Major refactor/restructure of project
- 🟡 Quarterly cleanup (every 3-4 months)

---

## Update Examples

### Example 1: Adding Security Agent (Simple Update)

**Before:** CLAUDE.md at 750 lines (🟢 Healthy)

**Changes needed:**
- Add `/security-phase` to slash commands (+3 lines)
- Update agent file list (+1 line)  
- Update workflow diagram (+2 lines)

**Action:** Option 1 (Simple Update)
**Result:** CLAUDE.md at 756 lines (still 🟢)

---

### Example 2: Major Feature Addition (Archival Recommended)

**Before:** CLAUDE.md at 950 lines (🟡 Warning)

**Changes needed:**
- New feature explanation (+50 lines)
- New slash commands (+20 lines)
- Workflow changes (+30 lines)

**Action:** Option 2 (Archive + Refresh)
**Result:** 
- Archived: CLAUDE-v3.md (950 lines)
- New: CLAUDE.md (520 lines)

---

### Example 3: Critical Bloat (Must Archive)

**Before:** CLAUDE.md at 1350 lines (🔴 Critical)

**Changes needed:**
- Minor updates (+10 lines)

**Action:** Option 2 (Archive + Refresh) - **MANDATORY**
**Result:**
- Archived: CLAUDE-v4.md (1350 lines)
- New: CLAUDE.md (480 lines)

---

## Post-Update Checklist

After updating CLAUDE.md:
- [ ] File size is under 800 lines (or freshly refreshed)
- [ ] All current slash commands documented
- [ ] All current agents listed
- [ ] Workflow diagrams reflect current process
- [ ] Tech stack is current
- [ ] No duplicate content (references docs instead)
- [ ] Archived version created (if Option 2)
- [ ] Archived version has header explaining archival

---

## When NOT to Update CLAUDE.md

**Don't update for:**
- Minor bug fixes (no structural changes)
- Test additions (doesn't affect project overview)
- Internal refactoring (no user-facing changes)
- Documentation updates in other files
- Constraint doc additions (just reference them)

**Only update when:**
- Project structure changes
- New major features added
- Workflow changes
- Tech stack changes
- New slash commands/agents
- File becomes bloated

---

## Special Case: Referencing Archived Versions

If user needs historical context:

```
USER: "How did we handle X in the past?"

RESPONSE:
"I can check the archived versions of CLAUDE.md for historical context.

Available archives:
- CLAUDE-v1.md (archived {date})
- CLAUDE-v2.md (archived {date})
- CLAUDE-v3.md (archived {date})

Which period are you interested in?"

THEN: Read relevant archive and extract historical context
```

---

**Remember:** CLAUDE.md should be a **concise entry point**, not comprehensive documentation. When in doubt, keep it short and reference other docs.