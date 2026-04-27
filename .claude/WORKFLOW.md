# Claude Code Instructions - Multi-Agent Implementation System

## Overview

This project uses a **multi-agent workflow system** for implementing features with high code quality and minimal context window bloat. Instead of doing everything in one long conversation, work is divided across specialized agents with clear handoffs.

## Three Workflow Modes

This system supports three modes of operation. Pick by the size and stakes
of the implementation in front of you.

### Mode 1: Orchestrator Mode (Single Session, Single Model)

**Use for:** Small/medium implementations (<500 lines, no compactions)

- One continuous Claude Code session
- Orchestrator manages all phases automatically
- Context accumulates within the session
- Fast and automatic with 3 human checkpoints
- Perfect for most day-to-day work

**Entry:** `/implement`. Final commit: `/commit #N`.

**When context is sufficient, this is the fastest way to work.**

See `docs/workflows/1-non-phased.md` for the full walkthrough.

### Mode 2: Phase Mode (Multi-Session, Single Model)

**Use for:** Large implementations (>500 lines, causing compactions)

- Separate fresh Claude Code session for each phase
- Manual control between phases (you close VS Code between each)
- Each phase gets a fresh 200K-token context
- No compactions, infinite iterations possible
- Maximum quality for complex single-model features

**Entry per phase:** `/implement-phase #N` → `/critic-phase #N` →
`/security-phase #N` (if needed) → `/test-phase #N` → `/commit-phase #N`.

**When you see "compaction occurred" messages, switch to this mode.**

See `docs/workflows/2-phased.md` for the full walkthrough.

### Mode 3: Hybrid Mode (Dual-Model, Recommended)

**Use for:** Production work, high-stakes features, anything where a
single-model blind spot would hurt

- Two models in the loop: Claude Code does `/scope`, `/build`, `/verify`;
  Codex (or another GPT-class model with file access) does `/architect`
  and `/review`
- The `issue-{N}.md` file in `.claude/implementations/` is the
  append-only coordination document between models
- Build → review loop runs until Codex stamps APPROVE; only two HITL
  points (`/scope` for alignment, `/verify` for the manual test plan)
- Catches blind spots that a single model would miss; spreads token
  usage across providers

**Entry:** `/scope #N` (Claude Code) → `/architect #N` (Codex) →
`/build #N` (Claude Code) → `/review #N` (Codex) → loop /build → /review
until APPROVE → `/verify #N` (Claude Code).

**The universal `/architect` and `/review` contract** lives in
`shared-docs/GPT_ARCHITECT_REVIEW_STANDARD.md`. It uses an "efficient
read pattern" — Codex loads only `## /architect` + `## Handover Contract`
when running `/architect`, and only `## /review` + `## Handover Contract`
when running `/review`. Project `AGENTS.md` files stay small (~120 lines)
and only carry project-specific overlays.

See `docs/workflows/3-hybrid.md` for the full walkthrough.

### How to Choose

```
Default → Hybrid Mode (Mode 3)
  ↑
  ├── Drop to Orchestrator Mode (Mode 1) if the change is small/contained
  │   and the dual-model overhead isn't worth it for this issue
  │
  └── Drop to Phase Mode (Mode 2) if the change is large enough that
      compactions are an issue but a single model is fine for the scope
```

**Rule of thumb:** Hybrid is the recommended default for serious work.
Orchestrator (single-session, single-model) is the right fall-back for
quick changes. Phase Mode (multi-session, single-model) is for big
single-model features where compactions would hurt quality.

## System Architecture

### Agent Files (The Workers)

Located in `.claude/agents/`:

1. **00-orchestrator-agent.md** - Manages the entire workflow, coordinates agents, handles checkpoints
2. **01-implementation-agent.md** - Writes code AND automated tests based on requirements
3. **02-critic-agent.md** - Reviews code for quality, correctness, maintainability
4. **03-security-agent.md** - Performs threat analysis and security review (conditional)
5. **04-testing-agent.md** - Runs tests, linting, type checking; validates implementation

### Command Files (How to Control the System)

Located in `.claude/commands/`:

**Single-Session Commands (Orchestrator Mode — Mode 1):**

- **implement.md** - Kickstarts workflow after planning
- **catchup.md** - Shows current workflow status
- **commit.md** - Final validation and commit

**Multi-Session Commands (Phase Mode — Mode 2):**

- **implement-phase.md** - Run Implementation Agent in fresh session
- **critic-phase.md** - Run Critic Agent in fresh session
- **security-phase.md** - Run Security Agent in fresh session
- **test-phase.md** - Run Testing Agent in fresh session
- **commit-phase.md** - Run Commit in fresh session

**Hybrid Commands (Hybrid Mode — Mode 3, dual-model):**

- **scope.md** - Phase 1 (Claude): scope + AskUserQuestion alignment + codebase exploration
- **build.md** - Phase 3 (Claude): implement architecture plan, write tests
- **verify.md** - Phase 5 (Claude): address review issues, run full validation, commit
- `/architect` and `/review` are not separate command files — they live as
  inline definitions in each project's `AGENTS.md` (overlay) on top of the
  shared `shared-docs/GPT_ARCHITECT_REVIEW_STANDARD.md` contract.

**Greenfield Inception:**

- **bmad-start.md** - 3-phase BMAD inception (Analyst → PM → Architect)
- **bmad-synthesize.md** - Synthesize BMAD output into the standard project
  docs (CLAUDE.md, AGENTS.md, manifest, business plan)

**Maintenance:**

- **update-claude.md** - Keep CLAUDE.md in sync with code changes

### Supporting Documentation

- **MANUAL-SESSION-CHECKLIST.md** - Step-by-step guide for multi-session workflow
- **.claude/docs/** - Critical constraint documentation (see below)

### Workflow State

- **Active implementations:** `{project}/.claude/implementations/issue-{number}.md`
- **Archived implementations:** `{project}/.claude/implementations/archive/issue-{number}.md`
- These are the **source of truth** during implementation
- All agents read from and write to these files
- Prevents context window bloat from repeated GitHub API calls
- **Archival:** Implementations automatically archived after successful commit

**Structure:**

```
{your-project}/.claude/
  └── implementations/
      ├── issue-456.md          # Active (in progress)
      ├── issue-457.md          # Active (in progress)
      └── archive/              # Completed implementations
          ├── issue-123.md      # Closed
          ├── issue-234.md      # Closed
          └── issue-345.md      # Closed
```

**When files are archived:**

- After successful `git commit`
- After GitHub issue is closed
- Automatically by commit command
- Moved from `implementations/` to `implementations/archive/`

---

## Security Agent Overview

### Purpose

The Security Agent performs deep threat modeling and security analysis on implementations. Unlike the Critic Agent (which focuses on code quality and architecture), the Security Agent thinks like an attacker to identify vulnerabilities.

### When Security Agent Runs

**Conditional Invocation (Orchestrator Mode):**
The Orchestrator automatically decides whether to invoke Security Agent based on changed files:

**HIGH-RISK changes (Security runs):**

- Authentication / authorization code (auth middleware, session handling)
- Payment / billing logic (subscriptions, checkout flows, payment-provider webhooks)
- API endpoints, especially public/external ones
- WebSocket / real-time features (connection auth, message handling)
- Database access pattern changes (admin/service-tier credentials, access policies)
- OAuth and federated-login flows
- Third-party integrations (new external services)
- File uploads or user-generated content

**LOW-RISK changes (Security skipped):**

- UI-only changes (styling, layout, pure components)
- Documentation updates
- Test files only
- Static assets

**Manual Invocation (Phase Mode):**
Run `/security-phase #123` after Critic phase when working with sensitive features.

### What Security Agent Checks

1. **OWASP Top 10 (2021)** - Systematic verification against industry-standard vulnerabilities
2. **Threat Modeling** - "How would an attacker exploit this?"
3. **Attack Scenarios** - Realistic attack paths specific to the implementation
4. **Security Patterns** - Auth, input validation, API security, payment security, WebSocket security
5. **Project-Specific Security** - Checks `.claude/docs/security-*.md` if exists

### Project-Specific Security Overlays

The Security Agent should also run any project-specific threat analysis declared
in the project's `AGENTS.md` (Project-Specific `/review` Inputs → checklist) and
in `.claude/docs/` constraint files. Examples of overlays a project might
declare:

- **Domain-data privacy threats** - Storage, transmission, and access control
  for sensitive content (private recordings, PII, health data, etc.)
- **Payment security** - Webhook signature validation, idempotency keys, usage-
  tracking manipulation
- **Real-time security** - WebSocket authentication, message injection,
  resource exhaustion
- **Federated-auth security** - OAuth flow integrity, session management, MFA
- **Domain-specific threats** - whatever attack surfaces are unique to the
  project's vertical

If you're a fork-and-run user, list your project's overlays in your project
`AGENTS.md` so Security Agent invocations include them automatically.

### Security Status Meanings

- **✅ SECURITY APPROVED** - No critical or important security issues. Safe to proceed to testing.
- **⚠️ SECURITY CONCERNS** - Important security issues found that should be addressed but aren't immediately exploitable.
- **❌ SECURITY BLOCKED** - Critical vulnerabilities that could lead to data breach, unauthorized access, or financial loss. Must fix before testing.

### Security Review Decision Guidelines

**ALWAYS run Security for:**

- ✅ Authentication/authorization changes
- ✅ Payment/billing logic
- ✅ API endpoints (especially public/external)
- ✅ WebSocket/real-time features
- ✅ Database access pattern changes
- ✅ Third-party integrations
- ✅ File uploads or user-generated content

**CAN SKIP Security for:**

- ❌ UI-only changes (styling, layout)
- ❌ Documentation updates
- ❌ Minor bug fixes (typos, display issues)
- ❌ Internal tools (not user-facing)
- ❌ Test files only

**When in doubt:** Run Security Phase. Better safe than breached.

**Orchestrator Mode vs Phase Mode Security:**

**Orchestrator Mode:**

- Security Agent invoked automatically based on file patterns
- Decision is transparent (logs "Security review triggered" or "Skipping security")
- No manual intervention needed

**Phase Mode:**

- You decide whether to run `/security-phase #123`
- Recommended after Critic for any sensitive features
- Can skip for obvious low-risk changes

---

## Constraint Documentation System

### The Docs Directory

**Location:** `.claude/docs/`

This directory contains documentation for the **most critical constraints** in the codebase. These are rules that, if violated, will break core functionality.

**Structure (illustrative — fill with your own constraints):**

```
.claude/docs/
├── README.md                       # Index with "when to read" triggers
├── 01-{constraint-name}.md         # e.g. data-pipeline separation rules
├── 02-{constraint-name}.md         # e.g. core algorithm parameters that must not change
├── 03-{constraint-name}.md         # e.g. display/render rules per content type
├── 04-{constraint-name}.md         # e.g. billing / payment invariants
├── 05-{constraint-name}.md         # e.g. identity / role mapping
├── 06-{constraint-name}.md         # e.g. data-isolation and access policies
├── 07-{constraint-name}.md         # e.g. usage limits / guardrail behavior
├── 08-{constraint-name}.md         # e.g. real-time/streaming architecture
└── EXTERNAL_API_CALLS.md           # API cost tracking reference (optional)
```

See `templates/constraint-doc.md.template` in this kit for the per-constraint
structure (rule statement, why it exists, good/bad code, common mistakes,
verification tests).

### When to Read These Docs

**The README.md in .claude/docs/ contains keyword triggers** that tell you which docs to read based on what you're implementing.

**Critic Agent automatically checks these docs** during review to verify constraint compliance.

**Security Agent checks these docs** for security implications of constraints.

**Example flow:**

```
User: "Add {feature touching a constrained area}"
↓
Hooks: "Read .claude/docs/README.md to check constraints"
↓
You read README, see matching keyword → read the relevant 0X-*.md constraint doc
↓
Implement following documented patterns
↓
Critic verifies compliance during review
↓
Security verifies security implications
```

**Key point:** These are comprehensive docs (300-400 lines each) with:

- Clear rule statements
- Historical context (why the rule exists)
- Good/bad code examples
- Common mistakes to avoid
- Verification tests

**With Opus 4.5, you WILL read and apply all of these thoroughly.** Don't skim.

---

## Hooks System

### What Are Hooks?

Hooks are automated scripts that run at specific points in Claude Code's workflow. They provide:

1. **Automated reminders** to check constraint docs
2. **Constraint validation** before code execution
3. **Sound notifications** when you're done or need input

### Hook Types

**1. PreToolUse Hooks**

- **Trigger:** Before Bash/CreateFile/StringReplace execute
- **Action:** Validates code against critical constraints
- **Why:** Catches violations BEFORE code is written (not during review)

**Examples (illustrative — wire your own constraints):**

```python
# You try to create this:
def store_user_secret(user_id, secret):
    public_table.insert({"user_id": user_id, "secret": secret})

# Hook catches it:
"ERROR: secrets must not be stored on a publicly-readable table.
 Use the secrets table with server-only access."
→ File creation BLOCKED

# You fix it:
def store_user_secret(user_id, secret):
    secrets_table.insert({"user_id": user_id, "secret": secret})

# Hook passes
→ File created successfully
```

**2. Sound Hooks (Optional)**

- **Stop:** Plays sound when you finish a task (no more watching!)
- **Notification:** Plays sound when you need input
- **Platform:** macOS only (easily adapted for Linux/Windows)

### Installing Hooks

**See:** `HOOK-INSTALLATION-GUIDE.md` for complete setup instructions.

**Quick start:**

1. Copy `global-hooks-settings.json` to `~/.claude/settings.json`
2. Copy hook scripts to `~/.claude/hooks/`
3. Make scripts executable
4. Start a new Claude Code session

**Hook scripts:**

- This kit does not bundle a constraint-validation hook script — those are
  inherently project-specific. If you want one, model it after a small Python
  or shell script that:
  - Reads the file/command being created or run
  - Greps for forbidden patterns derived from your `.claude/docs/` constraints
  - Emits a clear ERROR line and exits non-zero to block the action

### How Hooks Improve Quality

**Without hooks:**

```
Implement → Review → Find constraint violation → Fix → Review again
(Waste time finding issues during review)
```

**With hooks:**

```
Try to write code → Hook catches violation → Fix immediately → Implement successfully
(Catch issues at write-time, not review-time)
```

**Defense in depth:**

- Hooks catch obvious violations (write-time)
- Critic catches subtle violations (review-time)
- Security catches threat scenarios (security-time)
- Tests catch functional violations (test-time)

---

## Standard Workflow

### Phase 1: Planning (Use Claude Code's Built-in Planning)

**IMPORTANT: Plan in a separate session, then start fresh for implementation.**

```
Session 1 - Planning:
User: [Describes what they want to build]
You (Claude Code): Use your planning mode and research tools to:
- Understand requirements
- Research the codebase
- Create a detailed implementation plan
- Get user approval on the plan

User: [Approves plan]
User: [Notes plan file path OR copies the plan]
User: [Closes VS Code]

Session 2 - Implementation (FRESH):
User: [Opens fresh VS Code]
User: [References plan file OR pastes plan]
User: "/implement" (Mode 1) or "/implement-phase" (Mode 2) or
      "/scope" (Mode 3 — hybrid; this also creates the issue file
       and runs alignment via AskUserQuestion)
```

**Why fresh session?**

- Planning sessions often consume 100-150K tokens
- Starting implementation in same session = already bloated
- Fresh session = pristine 200K tokens for implementation
- Higher quality, fewer compactions

**Output:** A detailed plan with requirements, approach, acceptance criteria

---

### Phase 2: Implementation (Use Multi-Agent System)

#### ORCHESTRATOR MODE (Single Session)

**Recommended: Start in fresh session after planning**

##### Step 0: Session Break (RECOMMENDED)

```
After planning completes:
1. Note the plan file path (if Claude Code saved it)
   OR copy the approved plan
2. Close VS Code (end planning session)
3. Open fresh VS Code session
4. Reference file or paste plan into new session
```

##### Step 1: Initialize Implementation

```
User: [References plan file OR pastes approved plan]
User: "/implement"

You: Execute .claude/commands/implement.md
→ Creates GitHub issue from plan
→ Creates .claude/implementations/issue-{N}.md
→ Initializes workflow state
→ Hands off to Orchestrator Agent
```

##### Step 2: Orchestrator Manages Full Workflow

```
Orchestrator executes automatically:

1. Implementation Agent
   └─ Writes code based on requirements
   └─ Writes automated tests (Jest + React Testing Library)
   └─ Documents in markdown
   └─ Status: ✅ SUCCESS / ⚠️ PARTIAL / ❌ BLOCKED

2. Critic Agent
   └─ Reviews quality, architecture, constraints
   └─ Status: ✅ SUCCESS / ⚠️ PARTIAL / ❌ BLOCKED
   └─ CHECKPOINT 1: Human decides if issues warrant retry

3. Security Agent (CONDITIONAL)
   └─ IF high-risk changes detected:
      └─ Performs threat analysis
      └─ Checks OWASP Top 10
      └─ Status: ✅ APPROVED / ⚠️ CONCERNS / ❌ BLOCKED
      └─ CHECKPOINT 2: Human decides if security issues warrant retry
   └─ ELSE: Skips to Testing

4. Testing Agent
   └─ Runs automated tests written by Implementation Agent
   └─ Runs linting and type checking
   └─ Validates acceptance criteria
   └─ Status: ✅ SUCCESS / ⚠️ PARTIAL / ❌ BLOCKED
   └─ CHECKPOINT 3: Human decides if test results allow commit

5. Ready for Commit
   └─ User runs /commit #123
```

##### Decision Points (Checkpoints)

**Checkpoint 1: After Critic**

- ✅ SUCCESS → Continue (or skip to Security/Testing if implemented)
- ⚠️ PARTIAL → Decide: Proceed or Retry
- ❌ BLOCKED → Must retry

**Checkpoint 2: After Security (if invoked)**

- ✅ SECURITY APPROVED → Continue to Testing
- ⚠️ SECURITY CONCERNS → Decide: Proceed or Retry
- ❌ SECURITY BLOCKED → Must retry

**Checkpoint 3: After Testing**

- ✅ SUCCESS → Proceed to Commit
- ⚠️ PARTIAL (lint warnings) → Decide: Proceed, Auto-fix, or Retry
- ❌ BLOCKED (tests failed) → Must retry

##### Step 3: Finalize and Commit

```
User: "/commit #123"

You: Execute .claude/commands/commit.md
→ Final validation (security if ran, tests, types)
→ Check/update CLAUDE.md if needed
→ Document in GitHub issue
→ Git commit with proper message
→ Close GitHub issue

User: git push
Done! 🎉
```

---

#### PHASE MODE (Multi-Session)

**Use when implementations are >500 lines or causing compactions**

##### Complete Workflow

```
Session 0: Planning
User: [Plan feature]
You: [Create plan, get approval]
User: [Note file OR copy plan]
User: [Close VS Code]

Session 1 (FRESH): Implementation
User: [Reference file OR paste plan]
User: "/implement-phase"
You: Execute .claude/commands/implement-phase.md
     → Implementation Agent writes code
     → Documents in markdown
User: [Close VS Code]

Session 2 (FRESH): Critic Review
User: "/critic-phase #123"
You: Execute .claude/commands/critic-phase.md
     → Critic Agent reviews with fresh 200K tokens
     → Reports: ✅ ⚠️ ❌
User: [Decide: continue or retry]
User: [Close VS Code]

Session 3 (FRESH): Security Review (IF NEEDED)
User: "/security-phase #123"
You: Execute .claude/commands/security-phase.md
     → Security Agent analyzes threats with fresh 200K tokens
     → Reports: ✅ ⚠️ ❌
User: [Decide: continue or retry]
User: [Close VS Code]

Session 4 (FRESH): Testing
User: "/test-phase #123"
You: Execute .claude/commands/test-phase.md
     → Testing Agent validates with fresh 200K tokens
     → Reports: ✅ ⚠️ ❌
User: [Decide: proceed or retry]
User: [Close VS Code]

Session 5 (FRESH): Commit
User: "/commit-phase #123"
You: Execute .claude/commands/commit-phase.md
     → Finalization with fresh 200K tokens
User: git push
Done! 🎉

Each session: Fresh 200K tokens
Total overhead: ~2 minutes of closing/opening
Benefit: No compactions, infinite iterations, maximum quality
```

##### When to Skip Security Phase

In Phase Mode, Security is manual. Run `/security-phase #123` when:

- ✅ Auth/authorization changes
- ✅ Payment/billing logic
- ✅ API endpoints
- ✅ WebSocket/real-time features
- ✅ Database access patterns
- ✅ Third-party integrations

Skip Security Phase for:

- ❌ UI-only changes
- ❌ Documentation
- ❌ Minor bug fixes
- ❌ Test files only

**For complete multi-session workflow guide:** See `.claude/MANUAL-SESSION-CHECKLIST.md`

---

#### HYBRID MODE (Dual-Model)

**Use when:** Production work, high-stakes features, anything where a
single-model blind spot would hurt. Recommended default for serious work.

##### Complete Workflow

```
Session 0: Planning (optional — /scope handles alignment too)
User: [Plan feature]
You: [Create plan, get approval]
User: [Note file OR copy plan]
User: [Close VS Code]

Phase 1 (Claude Code, FRESH): /scope
User: [Reference plan file OR paste plan]
User: "/scope"
You: Execute .claude/commands/scope.md
     → AskUserQuestion alignment (THIS IS A HITL POINT)
     → Codebase exploration
     → Constraint check
     → Optional REQ-XX traceability tags
     → Create GitHub issue + .claude/implementations/issue-{N}.md
     → Write Original Requirements + Codebase Exploration sections
User: [Switch to Codex IDE]

Phase 2 (Codex): /architect #N
User: "/architect #N"
Codex:
     → Read project AGENTS.md
     → Partial-read shared GPT_ARCHITECT_REVIEW_STANDARD.md (## /architect + ## Handover Contract)
     → Read source files referenced in Codebase Exploration
     → Validate scope by exploring related code
     → Write "## Architecture Plan - Attempt N" to issue file
     → Recommendation: READY TO BUILD / NEEDS CLARIFICATION / REQUIREMENTS UNCLEAR
User: [Switch to Claude Code]

Phase 3 (Claude Code): /build #N
User: "/build #N"
You: Execute .claude/commands/build.md
     → Read architecture plan
     → Implement following plan, document any deviations
     → Write tests alongside implementation
     → Run quick smoke validation
     → Append "## Implementation Notes - Attempt N" to issue file
User: [Switch to Codex IDE]

Phase 4 (Codex): /review #N
User: "/review #N"
Codex:
     → Read project AGENTS.md
     → Partial-read shared GPT_ARCHITECT_REVIEW_STANDARD.md (## /review + ## Handover Contract)
     → Read ONLY the files listed in Implementation Notes (scope rule)
     → Run `git diff` only on those files
     → Write "## GPT Review - Attempt N" to issue file
     → Recommendation: APPROVE / REQUEST CHANGES / BLOCK

IF recommendation is REQUEST CHANGES or BLOCK:
  Loop /build → /review until APPROVE.
  Each new /review attempt starts with a Previous Issues Verification
  table marking each prior issue FIXED / STILL PRESENT / PARTIALLY FIXED.

IF recommendation is BLOCK and the issue needs redesign rather than
re-implementation, run /architect #N again rather than /build #N.

Phase 5 (Claude Code): /verify #N
User: "/verify #N"
You: Execute .claude/commands/verify.md
     → Address remaining Critical and Important issues from review
     → Run targeted Security Pre-Check (auth surface, secrets, rate limits)
     → Verify migrations are applied (if migrations changed)
     → Run full test suite, type check, lint
     → Optional browser-automation verification
     → Generate Manual Test Plan from acceptance criteria
     → Verify acceptance criteria
     → Write "## Verification & Commit" to issue file
     → THIS IS THE SECOND HITL POINT: user walks through manual plan
       in browser, signs off
     → Commit, optionally update CLAUDE.md/AGENTS.md/manifest, close
       GitHub issue, archive issue file

User: git push
Done! 🎉
```

Two HITL points only: `/scope` (alignment) and `/verify` (manual test
plan + sign-off). Everything else (architect → build → review loop) runs
without human intervention beyond switching IDEs at each phase boundary.

##### Hybrid Mode vs Phase Mode

Both involve switching contexts between phases. The difference:

- **Phase Mode** is the same model in fresh sessions. Solves
  context-bloat. Doesn't solve same-model blind spots.
- **Hybrid Mode** is two different models. Solves blind spots
  (Claude implements, Codex reviews). Token usage spreads across
  providers. IDE-switching overhead is comparable to Phase Mode.

**For complete hybrid walkthrough:** See `docs/workflows/3-hybrid.md`.
**For the universal `/architect` + `/review` contract:** See
`shared-docs/GPT_ARCHITECT_REVIEW_STANDARD.md`.

---

## Troubleshooting

### "I don't see the plan in context"

→ User needs to share the plan or provide issue number

### "GitHub issue creation failed"

→ Offer local-only mode or help fix GitHub access

### "Tests keep failing"

→ After 2-3 retries, suggest the approach might need rethinking

### "User asks to skip validation"

→ Warn about risks, but allow if they insist with "proceed anyway"

### "Unclear which mode to use"

→ Ask: "Do you have an issue number, or should I use the plan from our conversation?"

### "Context compactions occurring"

→ Switch to Phase Mode (multi-session workflow)
→ Each phase gets fresh 200K tokens

### "Lost track of which phase I'm in"

→ Run `/catchup #123` to see current workflow state

### "Phase commands not recognized"

→ Ensure CLAUDE.md includes slash command definitions
→ Or use: "Execute .claude/commands/implement-phase.md for issue #123"

### "Too many retries (3-4+ attempts)"

→ Consider breaking into smaller issues
→ Review approach - might need architectural rethink

### "Wrong command used"

→ Orchestrator commands (/implement) won't work in Phase Mode
→ Phase commands (/implement-phase) expect existing markdown
→ Use correct command for your workflow

### "Markdown file doesn't exist"

→ Phase commands expect markdown already exists
→ For NEW implementations: use `/implement` first
→ For resuming: verify issue number is correct

---

## Advanced Usage

### Resuming Old Work

```
User: "Implement issue #789" (from weeks ago)
You: Fetch from GitHub, create markdown, start fresh
```

### Multiple Issues in Parallel

```
Each issue has its own markdown file
Catchup command helps track multiple issues
Orchestrator is stateless - works independently per issue
```

### Custom Workflows

Users might deviate from standard flow:

- That's okay - be flexible
- Core principle: markdown file as state tracker
- Agents can be invoked in different orders if needed

---

## Meta-Notes

### What Makes This System Good

1. **Context Management**: Markdown files prevent token bloat
2. **Quality Gates**: Critic, Security, and Testing catch issues early
3. **Security First**: Automatic threat analysis for high-risk changes
4. **Flexibility**: Works for new implementations or resuming old ones
5. **Audit Trail**: Full history in markdown files
6. **Human-in-Loop**: Pauses at critical decision points (3 checkpoints)

### What This System Is Not

- Not for every task (overkill for small changes)
- Not fully autonomous (requires human decisions at checkpoints)
- Not a replacement for good planning (garbage in, garbage out)

### Evolution

This system will evolve:

- Agent instructions may be refined
- New agents might be added
- Workflows might be adjusted

Always check the actual .md files for latest instructions.

---

## Complete Command Reference

### Single-Session Commands (Orchestrator Mode)

**Use for small/medium implementations (<500 lines, no compactions):**

#### /implement

**File:** `.claude/commands/implement.md`
**Usage:** `/implement` or `/implement #123`
**What it does:**

- Extracts plan from context OR fetches from GitHub
- Creates issue and markdown file
- Invokes Orchestrator to manage full workflow
- Pauses at 3 checkpoints for human decisions (Critic, Security if applicable, Testing)

#### /catchup

**File:** `.claude/commands/catchup.md`
**Usage:** `/catchup #123`
**What it does:**

- Shows current workflow state
- Displays what's completed, what's pending
- Shows if Security was invoked
- Provides next step recommendations
- Read-only, no modifications

#### /commit

**File:** `.claude/commands/commit.md`
**Usage:** `/commit #123`
**What it does:**

- Final validation (security if ran, tests, types, criteria)
- Checks/updates CLAUDE.md if needed
- Documents in GitHub issue
- Git commit with proper message
- Closes GitHub issue

---

### Multi-Session Commands (Phase Mode)

**Use for large implementations (>500 lines, causing compactions):**

#### /implement-phase

**File:** `.claude/commands/implement-phase.md`
**Usage:** `/implement-phase #123`
**What it does:**

- Runs Implementation Agent in FRESH session
- Loads context from markdown
- Writes code and documents
- Reports completion
  **After:** User closes VS Code

#### /critic-phase

**File:** `.claude/commands/critic-phase.md`
**Usage:** `/critic-phase #123`
**What it does:**

- Runs Critic Agent in FRESH session
- Loads context from markdown
- Reviews code quality
- Reports status (✅ ⚠️ ❌)
  **After:** User decides next step and closes VS Code

#### /security-phase

**File:** `.claude/commands/security-phase.md`
**Usage:** `/security-phase #123`
**What it does:**

- Runs Security Agent in FRESH session
- Loads context from markdown
- Performs threat analysis and OWASP checks
- Reports status (✅ ⚠️ ❌)
  **When to use:** After Critic, before Testing (for auth/payment/API/WebSocket changes)
  **After:** User decides next step and closes VS Code

#### /test-phase

**File:** `.claude/commands/test-phase.md`
**Usage:** `/test-phase #123`
**What it does:**

- Runs Testing Agent in FRESH session
- Loads context from markdown
- Validates tests, linting, types
- Reports results
  **After:** User decides next step and closes VS Code

#### /commit-phase

**File:** `.claude/commands/commit-phase.md`
**Usage:** `/commit-phase #123`
**What it does:**

- Runs Commit in FRESH session
- Same validation as regular /commit
- Documents, commits, closes issue
  **After:** User runs `git push`

---

### Hybrid Commands (Hybrid Mode)

**Use for production work, high-stakes features, or anything where a
single-model blind spot would hurt.** Recommended default.

#### /scope

**File:** `.claude/commands/scope.md`
**Usage:** `/scope` or `/scope #N`
**Run in:** Claude Code
**What it does:**

- Invokes `AskUserQuestion` extensively for product/UX/scope alignment (HITL)
- Creates a GitHub issue (or local issue) and `.claude/implementations/issue-{N}.md`
- Reads the project manifest and CLAUDE.md
- Explores the codebase: relevant files, existing patterns, applicable
  constraints, security considerations
- Optionally tags acceptance criteria with REQ-XX traceability tags
- Writes Original Requirements + Codebase Exploration sections to the issue file
  **After:** User switches to Codex and runs `/architect #N`

#### /architect (Codex)

**Defined in:** project `AGENTS.md` (overlays) on top of
`shared-docs/GPT_ARCHITECT_REVIEW_STANDARD.md` (universal contract)
**Usage:** `/architect #N`
**Run in:** Codex
**What it does:**

- Partial-reads the shared standard (`## /architect` + `## Handover Contract`)
- Reads the source files referenced in Codebase Exploration
- Validates scope by exploring related code
- Writes `## Architecture Plan - Attempt N` to the issue file
- Final recommendation: READY TO BUILD / NEEDS CLARIFICATION / REQUIREMENTS UNCLEAR
  **After:** User switches to Claude Code and runs `/build #N`

#### /build

**File:** `.claude/commands/build.md`
**Usage:** `/build #N`
**Run in:** Claude Code
**What it does:**

- Reads the architecture plan from the issue file
- Implements following the plan, documents any deviations
- Writes tests alongside implementation
- Runs a quick smoke validation (not the full suite)
- Appends `## Implementation Notes - Attempt N` to the issue file
  **After:** User switches to Codex and runs `/review #N`

#### /review (Codex)

**Defined in:** project `AGENTS.md` (overlays) on top of
`shared-docs/GPT_ARCHITECT_REVIEW_STANDARD.md` (universal contract)
**Usage:** `/review #N`
**Run in:** Codex
**What it does:**

- Partial-reads the shared standard (`## /review` + `## Handover Contract`)
- Reads ONLY the files listed in Implementation Notes (scope rule — ignores
  unrelated working-tree changes from other issues)
- Runs `git diff` only on those files
- Writes `## GPT Review - Attempt N` to the issue file with categorized
  findings (🔴 Critical / 🟡 Important / 🟢 Minor), Security Analysis
  (OWASP + STRIDE + project checks), tests recommended, AI bias
  observations, REQ-XX traceability verification
- Final recommendation: APPROVE / REQUEST CHANGES / BLOCK
  **After:** If APPROVE, user runs `/verify #N` in Claude Code. Otherwise,
  user loops back to `/build #N` (or `/architect #N` if BLOCK indicates a
  redesign is needed).

#### /verify

**File:** `.claude/commands/verify.md`
**Usage:** `/verify #N`
**Run in:** Claude Code
**What it does:**

- Addresses Critical and Important issues from the review
- Runs the Security Pre-Check (auth surface, secrets, rate limits, sensitive
  API calls, financial controls)
- Verifies database migrations are applied to the dev DB (if migrations changed)
- Runs the full test suite, type check, lint
- Runs browser-automation verification (if available)
- Generates the Manual Test Plan from acceptance criteria (HITL — user walks
  through it in a browser and signs off)
- Verifies acceptance criteria
- Commits, optionally updates CLAUDE.md / AGENTS.md / manifest, closes the
  GitHub issue, archives the issue file from `.claude/implementations/`
  to `.claude/implementations/archive/`

---

### Hybrid Workflow Pattern

```
Phase 0: Planning (optional)         [Claude Code]
Phase 1: /scope #N                   [Claude Code, fresh] — HITL: alignment
         User: switch to Codex IDE
         ↓
Phase 2: /architect #N               [Codex]
         User: switch to Claude Code
         ↓
Phase 3: /build #N                   [Claude Code]
         User: switch to Codex IDE
         ↓
Phase 4: /review #N                  [Codex]
         IF NOT APPROVE:
           User: switch to Claude Code, loop /build → /review
         IF APPROVE:
           User: switch to Claude Code
         ↓
Phase 5: /verify #N                  [Claude Code] — HITL: manual test plan
         User: git push
         Done!

Two HITL points only: /scope and /verify.
Everything in between is autonomous; user is the courier between IDEs.
```

---

### Multi-Session Workflow Pattern

```
Session 0: Planning
Session 1: /implement-phase #123
           [Implementation works]
           User closes VS Code
           ↓
Session 2: /critic-phase #123
           [Critic reviews with fresh context]
           User decides: continue or retry
           User closes VS Code
           ↓
Session 3: /security-phase #123 (IF NEEDED)
           [Security analyzes with fresh context]
           User decides: continue or retry
           User closes VS Code
           ↓
Session 4: /test-phase #123
           [Testing validates with fresh context]
           User decides: proceed or fix
           User closes VS Code
           ↓
Session 5: /commit-phase #123
           [Finalization with fresh context]
           User: git push
           Done!

Each phase: 200K fresh tokens
Total overhead: ~2 minutes of closing/opening
Benefit: No compactions, infinite iterations
```

**For complete workflow guide:** See `.claude/MANUAL-SESSION-CHECKLIST.md`

---

## When to Use Which Mode: Decision Guide

### Choose Hybrid Mode (Mode 3 — Recommended Default) If:

- ✅ Production work, high-stakes features
- ✅ Quality is critical and rework cost is high
- ✅ You want a second model to review what the first model built
- ✅ You want to spread token usage across providers
- ✅ Touching auth, payments, real-time, AI cost-abuse surfaces

### Choose Orchestrator Mode (Mode 1 — Single Session) If:

- ✅ Implementation is <500 lines
- ✅ No previous compaction issues
- ✅ Want automatic workflow management
- ✅ Fast turnaround needed
- ✅ The dual-model overhead isn't worth it for this issue
- ✅ Comfortable with context accumulation

### Choose Phase Mode (Mode 2 — Multi-Session) If:

- ✅ Implementation is >500 lines
- ✅ Seeing "compaction occurred" warnings
- ✅ Previous implementations hit context limits
- ✅ Quality is critical (production code) but hybrid's IDE-switching
  isn't justified for this particular feature
- ✅ Need many iterations within a single model
- ✅ Complex feature with multiple aspects

### How to Tell You Need Phase Mode (vs staying in Orchestrator):

1. You see "context compacted" messages
2. Later agents (Testing) produce lower quality output
3. Implementation is obviously large (5+ files, 1000+ lines)
4. You've tried Orchestrator mode and quality degraded

### How to Tell You Want Hybrid (vs staying in Phase Mode):

1. You're seeing the same model approve its own implementation more
   easily than seems right
2. Tests written by Implementation Agent feel like they're confirming
   the implementation rather than verifying the requirement
3. The change touches a domain (auth, payments, real-time) where
   getting it wrong costs more than the IDE-switching overhead

**General rules:**

- Default to Hybrid for serious work
- Drop to Orchestrator for quick changes
- Drop to Phase Mode for big single-model features where compactions
  would hurt quality but the dual-model trade-off isn't worth it

---

## Quick Start Checklist

When user wants to implement something:

- [ ] Did they finish planning? (If no → plan first)
- [ ] Do they have an approved plan? (If no → get approval)
- [ ] Are they in a fresh session? (If no → recommend session break)
- [ ] Are they saying "Implement this" or giving issue number?
- [ ] Read appropriate command file (implement.md or implement-phase.md)
- [ ] Execute the command instructions
- [ ] Hand off to Orchestrator (or specific agent in Phase Mode)
- [ ] Guide user at checkpoints
- [ ] Finalize with Commit command

---

**Remember:** You're coordinating a team of specialized agents. Your job is to:

1. Understand which phase the user is in
2. Invoke the right command/agent
3. Follow the instructions in the .md files precisely
4. Guide the user at decision points
5. Maintain the markdown files as state trackers

**When in doubt:** Read the relevant .md file. The instructions are comprehensive and battle-tested.

Good luck! 🚀
