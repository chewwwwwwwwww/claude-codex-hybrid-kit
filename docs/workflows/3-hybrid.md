# Workflow 3: Hybrid (Dual-Model, Recommended)

## When to use

- Production features where rework cost is high
- Anything touching auth, payments, real-time, AI cost-abuse surfaces
- You want a second model to review what the first model built
- You want to spread token usage across providers

This is the recommended default for serious work. The phased and
non-phased modes are kept for different size/complexity points, but
hybrid is what most issues should run through.

## What runs

Five phases. Two models. The `issue-{N}.md` file in
`.claude/implementations/` is append-only and accumulates each
phase's output:

```
Phase 1: /scope #N                  (Claude Code)
  → AskUserQuestion alignment
  → Codebase exploration
  → Constraint check
  → Optional REQ-XX traceability tags
  → Writes:  Original Requirements
             Codebase Exploration
             Questions for Architect
  → Issue file Status: Scoping → Architecting

Phase 2: /architect #N              (Codex)
  → Reads project AGENTS.md
  → Partial-reads GPT_ARCHITECT_REVIEW_STANDARD.md (## /architect + ## Handover Contract)
  → Reads source files referenced in Codebase Exploration
  → Validates scope by exploring related code
  → Writes:  Architecture Plan - Attempt N
             (Approach, Implementation Steps, Data Model Changes,
              API Changes, Frontend Changes, Test Strategy, Risk
              Assessment, Constraint Compliance, Open Questions,
              Recommendation)
  → Recommendation: READY TO BUILD / NEEDS CLARIFICATION / REQUIREMENTS UNCLEAR

Phase 3: /build #N                  (Claude Code)
  → Reads Architecture Plan from the issue file
  → Implements following the plan
  → Writes tests alongside implementation
  → Documents deviations from the plan with rationale
  → Writes:  Implementation Notes - Attempt N
             (Files Changed, Tests Written, Key Decisions,
              For Reviewer, Security-Sensitive Areas)
  → Issue file Status: Implementing → Reviewing

Phase 4: /review #N                 (Codex)
  → Reads project AGENTS.md
  → Partial-reads GPT_ARCHITECT_REVIEW_STANDARD.md (## /review + ## Handover Contract)
  → Reads only the files listed in Implementation Notes (scope rule)
  → Runs git diff -- <those files only>
  → Writes:  GPT Review - Attempt N
             (Overall Assessment, 🔴 Critical, 🟡 Important, 🟢 Minor,
              Security Analysis (OWASP + STRIDE + project checks),
              Additional Tests Recommended, Frontend Flow Tests,
              Positive Highlights, AI Bias Observations,
              Requirement Traceability Verification, Recommendation)
  → Recommendation: APPROVE / REQUEST CHANGES / BLOCK

  IF not APPROVE: loop back to /build → /review

Phase 5: /verify #N                 (Claude Code)
  → Addresses Critical and Important issues from review
  → Runs targeted Security Pre-Check (auth surface, secrets, rate limits)
  → Verifies migrations are applied (if migrations changed)
  → Runs full test suite, type check, lint
  → Optional browser-automation verification
  → Generates the Manual Test Plan from acceptance criteria
  → Verifies acceptance criteria
  → Writes:  Verification & Commit
             (Issues Addressed, Issues Deferred, Test Results,
              Acceptance Criteria, Commit SHA)
  → Asks user: "Proceed with commit?"
  → Commits, optionally updates project docs, closes GitHub issue,
    archives issue file
  → Issue file Status: Reviewing → Complete
```

## Two HITL points only

The pipeline only needs human attention at two places:

1. **`/scope`** — Claude invokes `AskUserQuestion` extensively for
   product / UX / scope-boundary calls. This is the single most
   leveraged time you spend on the issue. Half an hour of clarifying
   here saves hours of rework.

2. **`/verify`** — Claude generates a manual test plan from
   acceptance criteria. You walk through the plan in a browser, sign
   off, commit.

Everything else (architect → build → review → review → review until
APPROVE) runs without human intervention beyond switching IDEs at
each phase boundary.

## Why two models

**Claim:** "Tests written by the same model that scores them is a
half-truth-but-real failure mode."

It's a half-truth because well-written tests genuinely do score
implementations correctly. It's also real because under pressure
(time budget, complex requirement) a single model will tend to
write tests that confirm the implementation it just wrote, rather
than the requirement that was asked for.

Switching models between `/build` (Claude Code) and `/review` (Codex)
breaks that loop. Each model is biased differently. Their blind
spots don't overlap perfectly. The build-review cycle exposes
issues that single-model workflows miss.

Secondary benefit: token usage spreads across providers. At volume,
that's a real cost reduction.

## The build → review loop

Most issues take 2–3 review attempts to reach APPROVE. The pattern:

```
/build  (attempt 1)        Implementation
/review (attempt 1)        REQUEST CHANGES — 2 important, 4 minor
/build  (attempt 2)        Address important; fix 2 of the 4 minors
/review (attempt 2)        REQUEST CHANGES — 1 important still standing
/build  (attempt 3)        Address remaining important
/review (attempt 3)        APPROVE
/verify                    Tests, manual plan, commit
```

Each `/review` attempt starts with a Previous Issues Verification
table that explicitly marks each prior issue as FIXED / STILL
PRESENT / PARTIALLY FIXED. The reviewer can't quietly drop concerns
between rounds.

`BLOCK` is reserved for issues that need redesign, not just
implementation rework. If `/review` returns BLOCK, you go back to
`/architect` rather than `/build`. That's the explicit semantic in
`GPT_ARCHITECT_REVIEW_STANDARD.md`.

## Scope rule for /review

`/review` only inspects files listed in **Implementation Notes** for
the current attempt. It does not flag unrelated working-tree changes
as scope creep. This matters when you're working on multiple issues
in parallel without branching — your working tree may have
uncommitted changes from issue #240 while you're reviewing issue
#243. The reviewer ignores #240's diffs entirely.

## Files involved

```
~/.claude/commands/
├── scope.md               # Phase 1 (Claude)
├── build.md               # Phase 3 (Claude)
└── verify.md              # Phase 5 (Claude)

shared-docs/                            # At parent of your projects
└── GPT_ARCHITECT_REVIEW_STANDARD.md    # Phases 2 & 4 (Codex reads partial)

{your-project}/
├── AGENTS.md              # Project overlay — Codex reads this first
├── CLAUDE.md              # Project conventions
├── docs/context/{ProjectName}.md       # Project manifest
└── .claude/implementations/
    └── issue-{N}.md       # State tracker (append-only)
```

## Trade-offs

**Strengths:**

- Two models = blind-spot coverage
- Token spread across providers
- Clear five-phase structure with append-only audit trail
- Two HITL points only — most of the loop is autonomous
- Build-review loop explicitly tracks attempts; reviewers can't
  quietly drop concerns
- Scope rule prevents review noise from parallel work

**Weaknesses:**

- IDE switching overhead. You're moving between Claude Code and
  Codex multiple times per issue.
- Two model bills. Spreading is a feature for cost predictability;
  it's a cost increase if you weren't paying for the second model
  before.
- The user is in the loop on every phase boundary today (typing
  the next command, switching IDEs). Orchestration above this
  workflow — automating the model-to-model handoff — is the
  obvious next step but isn't included in this kit.

## Move to phased when

The implementation is large enough that you'd rather have one model
focus deeply on one phase at a time, and the dual-model trade-off
isn't worth the IDE-switching overhead for this particular feature.

## Stay in hybrid when

Default. Make the user explicitly choose another mode for a reason.
