# BMAD Start — Project Inception

## Purpose

Walk through 3 BMAD-inspired planning phases to produce raw artifacts for a new project. Output goes to `{project}/_bmad-output/`.

## When to Use

- Starting a brand new project from scratch
- Not for features within existing projects (use `/scope` or `/implement`)

## Execution Instructions

When the user says "/bmad-start", execute these steps:

### Step 1: Initialize

```
ACTION: Confirm project name with user
ACTION: Create _bmad-output/ directory in project root

IF _bmad-output/ already exists:
  OUTPUT: "Found existing _bmad-output/ directory. Resume previous session or start fresh?"
  IF "fresh": Delete contents of _bmad-output/
  IF "resume": Check which phases are complete and resume from next

STORE: {project_name}
```

### Step 2: Phase 1 — Analysis (Analyst Role)

Adopt the mindset of a product analyst. Guide user through:

**Discovery Questions (ask iteratively, not all at once):**

- What problem are you solving? Who has this problem?
- What's the target market? (geography, demographics, niche)
- What existing solutions do users currently use? What's wrong with them?
- What's your unique angle or differentiator?
- What does success look like in 3 months? 12 months?

**Feature Brainstorm:**

- Core features (MVP — what must ship)
- Nice-to-have features (Phase 2)
- Future vision features (Phase 3+)

**Key Decisions to Capture:**

- Revenue model (freemium, subscription, transaction-based)
- Target platforms (web, mobile, both)
- Scale expectations (100 users? 10K? 1M?)

```
OUTPUT: Write _bmad-output/product-brief.md
FORMAT: Markdown with sections for Problem, Users, Market, Features (by phase),
        Key Decisions, Success Metrics
```

Confirm with user: "Product brief ready. Review it, then say 'next' for Planning."

### Step 3: Phase 2 — Planning (PM Role)

Adopt the mindset of a product manager. Read `_bmad-output/product-brief.md` as input.

**Requirements Definition:**

- For each MVP feature, write user stories:
  "As a {user}, I want to {action} so that {benefit}"
- For each user story, write acceptance criteria
- Identify critical constraints (business rules that MUST NOT be violated)

**Epic Organization:**

- Group user stories into epics (logical feature areas)
- Order epics by implementation priority
- Estimate relative complexity (S/M/L/XL)

**Constraint Identification:**

- Payment/financial rules
- Data privacy / regulatory requirements
- Authentication / authorization rules
- Third-party API limitations
- Domain-specific business logic

```
OUTPUT: Write _bmad-output/prd.md
FORMAT: Markdown with sections for Epics (each containing Stories + Acceptance Criteria),
        Constraints table, User Personas, Success Metrics
```

Confirm with user: "PRD ready. Review it, then say 'next' for Solutioning."

### Step 4: Phase 3 — Solutioning (Architect Role)

Adopt the mindset of a software architect. Read `_bmad-output/product-brief.md` and `_bmad-output/prd.md`.

**Tech Stack Decision:**

- Frontend: framework, language, styling
- Backend: framework, language, API style
- Database: type, provider, schema approach
- Auth: provider, strategy
- Hosting: provider, deployment strategy
- External services: payments, email, analytics, AI

**Architecture Design:**

- System diagram (describe in text, suggest Mermaid if useful)
- Data model (entities, relationships, key fields)
- API design (endpoints or routes, auth patterns)
- State management approach
- File/folder structure

**Implementation Readiness Check:**

- For each epic: Is it implementable with this stack? Any blockers?
- For each constraint: How is it enforced architecturally?
- External dependencies: Are APIs available? Are there rate limits?
- Security: How are the OWASP Top 10 addressed by design?

```
OUTPUT: Write _bmad-output/architecture.md
FORMAT: Markdown with sections for Tech Stack Table, Architecture Overview,
        Data Model, API Design, Folder Structure, Readiness Matrix, Security Design
```

Confirm with user: "Architecture ready. All 3 phases complete. Run `/bmad-synthesize` to produce your project documentation."

### Step 5: Report Completion

```
OUTPUT:
"BMAD INCEPTION COMPLETE

Artifacts created:
- _bmad-output/product-brief.md (Analysis)
- _bmad-output/prd.md (Planning)
- _bmad-output/architecture.md (Solutioning)

Next step: /bmad-synthesize
This will produce your standardized project docs:
  ✅ CLAUDE.md
  ✅ docs/context/{Project}.md (manifest)
  ✅ AGENTS.md
  ✅ docs/business-plan.md
  ✅ GitHub repo + milestones + issues"
```
