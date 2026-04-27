# Workflow 2: Phased (Multi-Session)

## When to use

- Implementations over ~500 lines of changed code
- You're seeing "context compacted" warnings in single-session mode
- The change is multi-faceted: several files, multiple subsystems,
  cross-cutting refactor
- Quality matters more than turnaround time
- You want each phase to get a full fresh-context budget

This is the right pick when you've outgrown non-phased but aren't ready
for the dual-model overhead of hybrid yet, or when the change is
single-model in nature but big.

## What runs

Each phase runs in its **own fresh Claude Code session**. You close
VS Code (or the equivalent) between phases. Each phase reads the
issue file, does its work, appends to the file, reports status. Next
session picks up where the previous one left off.

```
Session 0: Planning
  User: [plan the feature]
  Claude: [research, plan, get user approval]
  User: [save plan, close VS Code]

Session 1 (FRESH): /implement-phase #123
  Claude: [Implementation Agent writes code + tests]
  Claude: [appends Implementation Notes to issue file]
  User: [close VS Code]

Session 2 (FRESH): /critic-phase #123
  Claude: [Critic Agent reviews quality + constraints]
  Claude: [reports ✅ SUCCESS / ⚠️ PARTIAL / ❌ BLOCKED]
  User: [decide: continue or retry, close VS Code]

Session 3 (FRESH): /security-phase #123       (if needed)
  Claude: [Security Agent threat-models]
  Claude: [reports ✅ APPROVED / ⚠️ CONCERNS / ❌ BLOCKED]
  User: [decide: continue or retry, close VS Code]

Session 4 (FRESH): /test-phase #123
  Claude: [Testing Agent runs full suite, validates criteria,
           generates manual test plan]
  Claude: [reports ✅ / ⚠️ / ❌]
  User: [decide: proceed or fix, close VS Code]

Session 5 (FRESH): /commit-phase #123
  Claude: [final validation, commit, optional CLAUDE.md updates,
           close GitHub issue, archive issue file]
  User: [git push]
  Done.
```

Each session: full fresh context budget. Total overhead: ~2 minutes of
closing/opening VS Code. Benefit: no compactions, infinite iterations,
maximum quality per phase.

## Security Phase: when to run

Phased mode does **not** auto-invoke `/security-phase`. You decide.

**Run it for:**

- ✅ Auth/authorization changes
- ✅ Payment/billing logic
- ✅ API endpoints (especially public/external)
- ✅ WebSocket/real-time features
- ✅ Database access pattern changes
- ✅ Third-party integrations
- ✅ File uploads / user-generated content
- ✅ AI/LLM features (cost-abuse risk)
- ✅ Environment variable / secret changes
- ✅ Database schema changes
- ✅ Deployment configuration changes

**Skip for:**

- ❌ UI-only changes (styling, layout)
- ❌ Documentation updates
- ❌ Minor bug fixes (typos, display issues)
- ❌ Test files only

When in doubt, run it. Better safe than breached.

## Files involved

Same five agent files as non-phased mode, plus a different set of
entry commands:

```
~/.claude/commands/
├── implement-phase.md     # Run Implementation Agent in fresh session
├── critic-phase.md        # Run Critic Agent in fresh session
├── security-phase.md      # Run Security Agent in fresh session
├── test-phase.md          # Run Testing Agent in fresh session
└── commit-phase.md        # Run commit in fresh session

~/.claude/agents/          # Same as non-phased
└── ...

{your-project}/.claude/implementations/
└── issue-{N}.md           # State tracker accumulates each phase's output
```

The issue file is the entire coordination mechanism. Each phase
command reads it, finds the relevant prior section (Implementation
Notes for Critic, Implementation Notes + Critic Review for Security,
etc.), does its work, appends.

## Resuming

```
/catchup #123
```

Tells you what state issue #123 is in — which phases are done, what
status they reported, what the recommended next step is. Useful when
you've been away from a feature for a few days and don't remember.

## Trade-offs

**Strengths:**

- Each phase gets a full 200K-token budget — no compactions ever
- Quality per phase is maximal — Critic isn't fighting bloated
  context from Implementation
- Infinite iterations possible — no degradation from staying in the
  same session too long
- Suits big features that genuinely need the room

**Weaknesses:**

- Friction. Closing and reopening VS Code (or whatever IDE) between
  phases costs ~2 minutes of human time per phase. For 5-phase
  flows, that's 10 minutes of pure overhead.
- Still single-model. Same blind-spot risk as non-phased — Critic
  reviewing Implementation by the same model.
- More to remember. Which phase comes next? What did Critic say?
  `/catchup` helps but it's still mental overhead the non-phased
  mode doesn't have.

## Move to hybrid when

You're tolerating the friction of phased and you want the
same-model blind spots gone. The hybrid mode is roughly the same
amount of friction (you're switching IDEs anyway) but with the
dual-model upside.

## Move back to non-phased when

A feature turns out smaller than expected and you're halfway through
phased mode. Just `/commit-phase` early once Testing reports clean —
you're not committed to all 5 phases.
