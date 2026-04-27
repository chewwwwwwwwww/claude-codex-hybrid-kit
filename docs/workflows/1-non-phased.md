# Workflow 1: Non-Phased (Single-Session Orchestrator)

## When to use

- Implementations under ~500 lines of changed code
- You haven't been seeing context-compaction warnings in past sessions
- The change is contained: a few files, one feature surface, no
  cross-cutting refactor
- You want fast turnaround and don't mind context accumulating within
  a single session

This is the right starting point if you're new to the kit. It's also
the right pick for ~70% of day-to-day issues on a typical project.

## What runs

`/implement` kicks off an in-session orchestrator that runs four agents
in sequence, pausing at three human checkpoints:

```
1. Implementation Agent
   └─ Writes code AND automated tests
   └─ Documents work in .claude/implementations/issue-{N}.md
   └─ Status: ✅ SUCCESS / ⚠️ PARTIAL / ❌ BLOCKED

2. Critic Agent
   └─ Reviews code quality, architecture, constraint compliance
   └─ Status: ✅ SUCCESS / ⚠️ PARTIAL / ❌ BLOCKED
   └─ ━━ CHECKPOINT 1: human decides if issues warrant retry ━━

3. Security Agent (CONDITIONAL)
   └─ Runs IF the changed files match high-risk patterns
   └─ Auth, payments, API endpoints, real-time, DB access policies, etc.
   └─ OWASP Top 10 + STRIDE + project-specific checks
   └─ Status: ✅ APPROVED / ⚠️ CONCERNS / ❌ BLOCKED
   └─ ━━ CHECKPOINT 2: human decides if security findings warrant retry ━━

4. Testing Agent
   └─ Runs the test suite, type-check, lint
   └─ Validates acceptance criteria
   └─ Status: ✅ SUCCESS / ⚠️ PARTIAL / ❌ BLOCKED
   └─ ━━ CHECKPOINT 3: human decides if test results allow commit ━━

5. /commit (separate command)
   └─ Final validation, commit message, optional CLAUDE.md doc updates,
      close GitHub issue, archive issue file
```

Security Agent invocation is automatic. The orchestrator inspects the
files Implementation Agent changed and skips Security entirely for
purely UI / docs / test-only changes. Override by running `/security-
phase #N` manually if you want it anyway.

## Entry

```
# In a fresh Claude Code session, after planning has produced an
# approved plan:

/implement                                  # plan in current context
/implement - use plan from .claude/plan.md  # plan in a file
/implement #123                              # resume an existing issue
```

The fresh-session recommendation matters. If you've spent hours
planning, your context is bloated. Close VS Code, reopen, paste or
reference the plan, run `/implement`. The implementation phase gets a
clean budget.

## Files involved

```
~/.claude/commands/
├── implement.md           # Entry point (you call this)
├── catchup.md             # Optional: shows current state
└── commit.md              # Final commit (you call this when done)

~/.claude/agents/
├── 00-orchestrator-agent.md    # Manages the workflow
├── 01-implementation-agent.md  # Writes code + tests
├── 02-critic-agent.md          # Reviews quality + constraints
├── 03-security-agent.md        # Threat-models high-risk changes
└── 04-testing-agent.md         # Runs tests + validates criteria

{your-project}/.claude/implementations/
└── issue-{N}.md           # State tracker (agents read & append)
```

## Trade-offs

**Strengths:**

- Fast — one session, no IDE switching, no manual phase-by-phase invocation
- Clear gates at three checkpoints
- Conditional Security saves time on low-risk changes
- Same model the whole way through (consistency)

**Weaknesses:**

- Context accumulates across all four agents in one session. By the
  time Testing Agent runs, the budget is partly spent on prior agents'
  output.
- Implementation Agent and Critic Agent are the same model. Not a
  problem for code-quality issues, but if the implementation has a
  systemic blind spot, the critic from the same model is unlikely to
  catch it.
- Security Agent uses the conditional invocation logic, which is
  good, but the patterns it matches against are heuristics. For
  truly sensitive features, prefer phased mode and run
  `/security-phase` manually.

## Move to phased when

You start seeing "context compacted" messages, or you notice that
Testing Agent's output quality drops compared to Implementation
Agent's earlier in the same session. Both are signals that the
single-session context is the bottleneck.

## Move to hybrid when

The same-model blind spot starts mattering. For production features
where rework cost is high, the dual-model loop catches things both
the implementation and review of a single model would miss.
