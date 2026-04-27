# Manual Multi-Session Workflow Checklist

Use this checklist when running large implementations that cause context compactions.

---

## When to Use Manual Sessions

✅ **Use manual multi-session workflow when:**
- Implementation is large (>500 lines)
- You see "compaction" messages in Claude Code
- Previous attempts hit context limits
- You want maximum quality (fresh context per phase)

❌ **Use regular /implement (Orchestrator mode) when:**
- Implementation is small (<500 lines)
- No compactions occurring
- Want automatic workflow

---

## The Workflow

### Step 0: Planning Session (SEPARATE)
```
[ ] Session 1 - Planning:
[ ] Discuss and plan feature in Claude Code
[ ] Get plan approved
[ ] Note the plan file path (Claude Code usually saves it)
    OR copy the approved plan (Ctrl+C / Cmd+C)
[ ] Close VS Code completely

Why separate? Planning often uses 100-150K tokens.
Starting implementation fresh = pristine 200K tokens.
```

### Step 1: Implementation Phase (FRESH SESSION)
```
[ ] Open VS Code / Claude Code (fresh session)
[ ] EASIEST: Reference the plan file
    "/implement-phase - use plan from .claude/plan-[filename].md"
    OR paste the approved plan
[ ] Run command: /implement-phase
[ ] Wait for Implementation Agent to complete
[ ] Note the new issue number (e.g., #123)
[ ] Note the status: ✅ SUCCESS / ⚠️ PARTIAL / ❌ BLOCKED
[ ] Close VS Code completely
```

**Next step:**
- If ✅ or ⚠️ → Go to Step 2 (Critic)
- If ❌ → Review blocker, resolve, retry Step 1

---

### Step 2: Critic Phase (FRESH SESSION)
```
[ ] Open VS Code / Claude Code (fresh session)
[ ] Run command: /critic-phase #123
[ ] Wait for Critic Agent to review
[ ] Note the status and recommendation
[ ] Close VS Code completely
```

**Next step:**
- If ✅ SUCCESS → Go to Step 2.5 (Security - if needed)
- If ⚠️ PARTIAL → Decide:
  - Issues acceptable → Go to Step 2.5 (Security - if needed)
  - Issues need fixing → Go back to Step 1 (retry)
- If ❌ BLOCKED → Go back to Step 1 (retry)

---

### Step 2.5: Security Phase (CONDITIONAL - FRESH SESSION)
```
[ ] Determine if security review needed:
    - Auth/payment/API/WebSocket changes? → YES
    - UI-only or documentation? → NO

IF NEEDED:
[ ] Open VS Code / Claude Code (fresh session)
[ ] Run command: /security-phase #123
[ ] Wait for Security Agent to analyze
[ ] Note the status
[ ] Close VS Code completely

IF NOT NEEDED:
[ ] Skip to Step 3 (Testing)
```

**Next step:**
- If ✅ SECURITY APPROVED → Go to Step 3 (Testing)
- If ⚠️ SECURITY CONCERNS → Decide:
  - Concerns acceptable → Go to Step 3
  - Security issues need fixing → Go back to Step 1 (retry)
- If ❌ SECURITY BLOCKED → Go back to Step 1 (retry)

---

### Step 3: Testing Phase (FRESH SESSION)
```
[ ] Open VS Code / Claude Code (fresh session)
[ ] Run command: /test-phase #123
[ ] Wait for Testing Agent to validate
[ ] Note the status
[ ] Close VS Code completely
```

**Next step:**
- If ✅ SUCCESS (no warnings) → Go to Step 4 (Commit)
- If ⚠️ PARTIAL (lint warnings) → Decide:
  - Warnings acceptable → Go to Step 4
  - Auto-fix linting → Stay in Step 3, run "auto-fix linting"
  - Manual fix → Go back to Step 1 (retry)
- If ❌ BLOCKED (tests failed) → Go back to Step 1 (retry)

---

### Step 4: Commit Phase (FRESH SESSION)
```
[ ] Open VS Code / Claude Code (fresh session)
[ ] Run command: /commit-phase #123
[ ] Review final summary
[ ] Confirm: "yes"
[ ] Handle CLAUDE.md update if prompted
[ ] Wait for commit to complete
[ ] Note commit SHA
[ ] Push: git push origin {branch}
```

**Done! 🎉**

---

## Quick Reference Card

**Copy this and keep handy:**

```
┌──────────────────────────────────────────────┐
│   MANUAL MULTI-SESSION WORKFLOW              │
├──────────────────────────────────────────────┤
│ Session 0: Plan → Note file → CLOSE          │
│ Session 1: /implement-phase - use plan from  │
│            [file] OR paste → CLOSE            │
│ Session 2: /critic-phase #123 → CLOSE        │
│ Session 3: /security-phase #123 → CLOSE      │
│            (if auth/payment/API/WebSocket)   │
│ Session 4: /test-phase #123 → CLOSE          │
│ Session 5: /commit-phase #123 → git push     │
│                                              │
│ Each session = Fresh 200K tokens            │
│ No compactions = High quality throughout    │
│ CRITICAL: Break after planning!             │
│ TIP: Security for sensitive features only!  │
└──────────────────────────────────────────────┘
```

---

## Handling Retries

### If Critic Finds Issues (Common)
```
Session 1: /implement-phase #123 → Attempt 1 → CLOSE
Session 2: /critic-phase #123 → "⚠️ Fix issues" → CLOSE
Session 3: /implement-phase #123 → Attempt 2 → CLOSE
Session 4: /critic-phase #123 → "✅ Good now" → CLOSE
Session 5: /test-phase #123 → CLOSE
Session 6: /commit-phase #123 → Done
```

### If Tests Fail (Less Common)
```
Session 1: /implement-phase #123 → CLOSE
Session 2: /critic-phase #123 → CLOSE
Session 3: /test-phase #123 → "❌ 2 tests failing" → CLOSE
Session 4: /implement-phase #123 → Attempt 2 → CLOSE
Session 5: /critic-phase #123 → CLOSE
Session 6: /test-phase #123 → "✅ Pass" → CLOSE
Session 7: /commit-phase #123 → Done
```

### Multiple Iterations (Complex)
```
Just keep alternating:
- /implement-phase → fixes/improvements
- /critic-phase → review
- (repeat until Critic says ✅)
- /test-phase → validation
- /commit-phase → finalize
```

---

## Time Estimates

| Phase | Typical Duration | Notes |
|-------|-----------------|-------|
| Implementation | 15-45 min | Varies by complexity |
| Critic | 5-15 min | Code review time |
| Testing | 5-20 min | Depends on test suite |
| Commit | 2-5 min | Quick finalization |
| **Session overhead** | ~30 sec each | Closing/opening VS Code |

**Total for simple path (no retries):** ~30-85 minutes + ~2 min overhead

---

## Tips for Success

### 1. Use Descriptive Issue Titles
Good: "Add PDF export with custom styling and metadata"
Bad: "Feature #123"

Why: Commit messages use issue titles

### 2. Track Your Progress
Keep notes:
```
Issue #123: PDF Export
- [ ] Implement phase (Attempt 1)
- [ ] Critic phase
- [ ] Implement phase (Attempt 2) - fixed error handling
- [ ] Critic phase - ✅ approved
- [ ] Test phase - ✅ passed
- [ ] Commit phase - ✅ done, SHA: abc123
```

### 3. Read the Markdown Between Phases
Check `.claude/implementations/issue-123.md` to see:
- What was done
- What feedback was given
- What needs fixing

### 4. Don't Skip Phases
Every phase has a purpose:
- Implementation: Writes code
- Critic: Catches issues before testing
- Testing: Validates everything works
- Commit: Finalizes and documents

### 5. Embrace Iteration
It's NORMAL to iterate 2-3 times:
- Attempt 1: First try
- Attempt 2: Fix Critic feedback
- Attempt 3: Fix any test issues

Each iteration gets FRESH context!

---

## Comparison: Single Session vs Multi-Session

### Single Session (Orchestrator)
```
/implement
[All phases in one session]
[Context accumulates]
[May compact on large implementations]

Time: Faster (no session overhead)
Quality: Good for small implementations
Context: Accumulates (can degrade)
```

### Multi-Session (Phase Commands)
```
/implement-phase #123 → CLOSE
/critic-phase #123 → CLOSE
/test-phase #123 → CLOSE
/commit-phase #123 → Done

Time: Slightly slower (~2 min overhead total)
Quality: Excellent (fresh context each phase)
Context: Always fresh (no degradation)
```

**Use multi-session for quality-critical or large implementations!**

---

## Troubleshooting

**Problem:** "Command not recognized"
**Solution:** Make sure slash commands are defined in CLAUDE.md. Use full command: "Execute .claude/commands/07-implement-phase.md for issue #123"

**Problem:** "Cannot find markdown file"
**Solution:** Check issue number. For NEW implementations, use /implement first.

**Problem:** "Session feels slow"
**Solution:** This is normal - you're running in fresh sessions. The quality is worth the ~30 second overhead.

**Problem:** "Lost track of where I am"
**Solution:** Run /catchup #123 to see current state.

**Problem:** "Too many retries"
**Solution:** After 3-4 attempts, review the approach. Might need to break into smaller issues.

---

## Success Criteria

You know it worked when:
- ✅ No compactions occurred during workflow
- ✅ High quality code throughout
- ✅ All tests passing
- ✅ GitHub issue closed with full documentation
- ✅ Commit is clean and complete
- ✅ CLAUDE.md stayed up to date

**That's the power of fresh context per phase!** 🚀