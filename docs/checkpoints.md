# The Two HITL Points (and the build-review loop)

In the hybrid workflow, the pipeline only needs human attention at two
phases: `/scope` and `/verify`. Everything else (`/architect`, `/build`,
`/review`) runs autonomously.

This doc explains what to do at each HITL point, what the autonomous
loop does in between, and what to watch for.

---

## HITL Point 1: `/scope`

**Where:** Phase 1, in Claude Code, before any code is written.

**What you do:** Answer Claude's `AskUserQuestion` prompts. Decide
product intent, UX preferences, scope boundaries, acceptance
criteria. Push back on assumptions you don't agree with.

**What Claude is doing:** Reading the project manifest and CLAUDE.md
to understand the codebase. Searching for existing implementations of
similar features. Reading constraint docs that the feature touches.
Identifying open questions that you (the human) can answer right now
versus questions that require architectural reasoning (those go to
the architect).

**Why this is the leveraged point:**

Half an hour of clarifying here saves hours of rework downstream.
The pattern that always burns time is: skip alignment at scope time,
get a beautiful architecture plan from `/architect`, get a clean
implementation from `/build`, then realize at `/verify` that the UX
implication of the originally-scoped behavior is wrong. Now you're
re-architecting from scratch.

If you're going to be lazy at any phase, don't let it be this one.

**Output of /scope:**

- A GitHub issue (if `gh` is available) or a local-only issue
- `.claude/implementations/issue-{N}.md` with:
  - Original Requirements (REQ-XX tagged if 5+ acceptance criteria)
  - Codebase Exploration (relevant files, existing patterns, key
    code snippets, applicable constraints, security considerations,
    questions for the architect)
- Status: Scoping → Architecting

**When to extend scope time:**

- The requirements are vague. Push back. Don't let a vague brief
  become Codex's problem at `/architect`.
- The feature crosses a constraint boundary. Make sure the
  constraint applies the way you think it does before architecting
  around it.
- The UX is unfamiliar to you. You'll catch fewer downstream
  surprises if you sketch the user flow now (in your head or on
  paper) before architecting.

---

## The Autonomous Loop: `/architect → /build → /review`

**Where:** Phases 2, 3, 4 — back and forth between Codex and Claude
Code until the reviewer stamps APPROVE.

**What you do:** Type the next command and switch IDE. That's it.
The phase-boundary input from you is roughly: "I'm done with
`/architect`, now I run `/build` in Claude Code." If you wanted to
automate even this, you'd be building the orchestration layer
mentioned in [docs/journey.md](journey.md). For now, you're the
human courier.

**What the loop is doing:**

1. **Codex /architect** — Reads project AGENTS.md, partial-reads the
   shared `## /architect` section, reads the source files referenced
   in Codebase Exploration, does its own exploration to validate
   scope, writes `## Architecture Plan - Attempt N` to the issue
   file. Recommendation: READY TO BUILD / NEEDS CLARIFICATION /
   REQUIREMENTS UNCLEAR.

2. **Claude /build** — Reads the architecture plan, implements
   following it (documenting any deviations), writes tests
   alongside, runs a quick smoke test, appends `## Implementation
Notes - Attempt N` with the files changed, tests written, key
   decisions, and a "For Reviewer" section flagging anything that
   needs special attention.

3. **Codex /review** — Reads only the files listed in Implementation
   Notes (the scope rule). Runs `git diff` only on those files.
   Partial-reads the shared `## /review` section. Writes
   `## GPT Review - Attempt N` with categorized findings (🔴
   Critical / 🟡 Important / 🟢 Minor), Security Analysis (OWASP +
   STRIDE + project-specific checks), Additional Tests Recommended,
   AI Bias Observations, and a Recommendation: APPROVE / REQUEST
   CHANGES / BLOCK.

4. **If APPROVE:** loop exits, you proceed to `/verify`.
   **If REQUEST CHANGES:** you run `/build` again. Claude addresses
   the scoped issues, appends `Implementation Notes - Attempt N+1`.
   Then `/review` again — the new attempt starts with a Previous
   Issues Verification table marking each prior issue as FIXED /
   STILL PRESENT / PARTIALLY FIXED.
   **If BLOCK:** the issue needs redesign. Run `/architect` again
   instead of `/build`.

**Typical attempt counts:**

- Simple features: APPROVE on attempt 1
- Medium features: APPROVE on attempt 2 or 3
- Complex features: 3+ attempts is normal. If you hit 5+, the
  scoping was probably under-specified — going back to `/scope` is
  often faster than continuing to grind on `/review` findings.

**What to watch for during the loop:**

- **A reviewer that always returns APPROVE.** If Codex consistently
  approves on attempt 1 with no findings, the reviewer isn't doing
  its job. Cross-check by manually inspecting a few `/build`
  outputs against your own eyes. If quality is genuinely high, fine
  — but if the reviewer is rubber-stamping, you've lost half the
  hybrid mode's value.
- **A reviewer that endlessly finds new issues.** If each `/review`
  attempt finds _different_ issues than the last (rather than
  verifying-and-closing previous ones), the reviewer has lost the
  thread. Read the Previous Issues Verification table — it should
  show prior issues being FIXED, not just disappearing from the
  current attempt's output.
- **Architecture-vs-implementation mismatch.** If the implementation
  consistently deviates from the plan and the deviations are
  legitimate, the architect is over-specifying. If the deviations
  aren't legitimate, the implementer is freelancing. Either way, a
  pattern across multiple issues is a signal to recalibrate.

---

## HITL Point 2: `/verify`

**Where:** Phase 5, in Claude Code, after `/review` returns APPROVE.

**What you do:**

1. Wait for Claude to finish addressing any remaining important
   issues from the review (it does this automatically).
2. Wait for Claude to run the test suite, type-check, lint.
3. **Walk through the Manual Test Plan** in a browser. Claude
   generates a per-issue plan from the acceptance criteria —
   numbered steps, edge cases auto-included based on what changed
   (i18n files, list/grid components, forms, layout/CSS,
   subscription gates).
4. Report any issues. Claude fixes them and regenerates the plan.
5. When the plan passes, reply "yes" to the commit prompt.

**What Claude is doing:**

- Addressing review findings (Critical mandatory, Important
  conditional, Minor optional)
- Running a Security Pre-Check (auth surface, client-side secrets,
  rate limits, sensitive API calls, financial controls)
- Verifying database migrations are applied to the dev DB (if
  migrations changed)
- Running the full test suite, type-check, lint
- Running browser-automation verification (if a browser MCP is
  available and the project's dev server is running)
- Generating the Manual Test Plan from acceptance criteria
- Verifying acceptance criteria
- Producing a Pre-Commit Review summary
- Committing once you sign off
- Optionally updating CLAUDE.md / AGENTS.md / project manifest if
  the change has documentation implications
- Closing the GitHub issue (if applicable)
- Archiving the issue file from `.claude/implementations/` to
  `.claude/implementations/archive/`

**Why this is the leveraged point:**

The Manual Test Plan exists because automated tests verify code
behavior, not feature behavior. A button can be wired up correctly
in code but say the wrong copy. A form can submit successfully but
the wrong toast can render. The manual plan is your last chance
to catch UX bugs before they ship.

**When to extend verify time:**

- The change is high-stakes (auth, payments, data integrity).
  Walk through the manual plan twice with different test
  accounts.
- The Security Pre-Check flagged anything 🟡 IMPORTANT. Decide
  whether to fix-now or defer with documentation.
- Migrations are involved. Verify the migration applied cleanly
  and that no follow-on data fix is needed.

---

## What's NOT a HITL point

- `/architect` — you don't review the architecture plan in detail.
  If it's wrong, `/review` will catch the consequences. If `/review`
  also misses them, that's a workflow gap to fix at the
  `GPT_ARCHITECT_REVIEW_STANDARD.md` level, not per-issue.
- `/build` — you don't review the diff in detail at this phase.
  `/review` is the diff review. Save your attention for the manual
  test plan at `/verify`.
- `/review` — you don't second-guess the reviewer per finding. If
  it says REQUEST CHANGES, run `/build` again. If it says APPROVE,
  proceed.

The discipline is: trust the process. The two HITL points are
load-bearing for a reason. The rest of the loop is load-bearing for
a different reason — its job is to apply consistent rigor without
your attention. Don't accidentally insert yourself in the middle of
the autonomous loop or you'll lose the gains it provides.
