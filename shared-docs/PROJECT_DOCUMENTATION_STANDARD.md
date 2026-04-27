# Project Documentation Standard

Standard documentation structure for all projects under `/VS`. Ensures consistency across codebases and enables AI agents to navigate any project with the same mental model.

**Reference implementations:** ChickenDuck, Imago, Genaurasity

---

## The Four Core Documents

Every project has up to four documentation files, each with a distinct purpose and audience:

### 1. `CLAUDE.md` — Quick Reference (~100-150 lines)

**Location:** Project root
**Audience:** AI coding agents (Claude, Copilot) and developers needing fast context
**Purpose:** Get productive in the codebase within 30 seconds

**Required sections:**

| #   | Section                            | Content                                                                |
| --- | ---------------------------------- | ---------------------------------------------------------------------- |
| -   | Header                             | Pointers to manifest, business plan, AGENTS.md                         |
| 1   | Project at a Glance                | 1-liner, stack summary, key features, deploy targets, link to manifest |
| 2   | Critical Constraints Quick Trigger | Table: trigger keywords → manifest section → core rule                 |
| 3   | Project-Specific Conventions       | Only overrides/extensions to global `~/.claude/CLAUDE.md` conventions  |
| 4   | Common Patterns                    | Table of reference files for canonical patterns                        |
| 5   | Development Workflow               | Setup, running locally, common tasks                                   |
| 6   | Multi-Agent Workflow               | 5-phase list + pointer to AGENTS.md                                    |

**Rules:**

- No business logic explanations — point to manifest sections instead
- No full constraint details — trigger table with one-line rules only
- No file trees — those live in the manifest
- Keep under 150 lines; if it's growing, content probably belongs in the manifest
- **Do not duplicate** conventions from the global `~/.claude/CLAUDE.md` (TypeScript, React, Python, file naming, DRY/SOLID). Only include project-specific overrides (e.g., project-specific formatter, state management library, styling approach)

### 2. `docs/context/{ProjectName}.md` — Technical Manifest (~300-500 lines)

**Location:** `docs/context/`
**Audience:** AI agents needing deep domain understanding, developers onboarding
**Purpose:** Single source of truth for all static technical facts about the project

**Required sections:**

| #   | Section                       | Content                                                               |
| --- | ----------------------------- | --------------------------------------------------------------------- |
| 1   | Project Overview              | Problem, solution, target users, current stage                        |
| 2   | Tech Stack                    | Component table with technology + notes; external services table      |
| 3   | Critical Constraints          | Numbered constraint table with rule + impact                          |
| 4   | Project Structure             | Full directory tree with annotations                                  |
| 5   | Key Concepts & Domain Logic   | Core entities, workflows, algorithms, business rules                  |
| 6   | API Endpoints                 | Method + path + description table                                     |
| 7   | Database Schema               | Full SQL DDL with types, constraints, RLS policies, triggers, indexes |
| 8   | Key Files                     | Table mapping files to purposes                                       |
| 9   | Known Issues & Technical Debt | Current limitations, debt, mistakes to learn from                     |
| 10  | External Resources            | Documentation links table                                             |

**Rules:**

- No business content (revenue, GTM, competitors, funding) — that belongs in the business plan
- Section 5 is the most project-specific — subsections vary by domain
- Schema section should include actual SQL DDL, not just entity sketches
- Keep sections self-contained — each should be readable without context from others

### 3. `AGENTS.md` — AI Agent Workflow Guide

**Location:** Project root
**Audience:** GPT (or other non-Claude AI) acting as architect/reviewer
**Purpose:** Define the multi-agent workflow plus the project-specific overlays for the shared GPT architect/review contract

**Required sections:**

- Role definition (which agent does what)
- Pointer to shared spec: `/VS/docs/GPT_ARCHITECT_REVIEW_STANDARD.md`
- Project context pointers (to manifest + CLAUDE.md)
- Key facts to internalize (critical constraints summary)
- Project-specific `/architect` inputs and architecture emphasis
- Project-specific `/review` inputs and checklist
- Handover deviations (only if the project differs from the shared standard)

**Shared command spec:**

- `/VS/docs/GPT_ARCHITECT_REVIEW_STANDARD.md` is the canonical cross-project definition of `/architect` and `/review`
- The shared spec owns:
  - attempt handling
  - output templates
  - REQ-XX traceability
  - STRIDE / security sections
  - AI bias observations
  - append-only handover rules

**Rules:**

- Constraint table should match manifest §3 (same numbering)
- Project context pointers should reference both manifest and CLAUDE.md
- Do not duplicate the full universal `/architect` and `/review` contract in every project `AGENTS.md`
- Keep `AGENTS.md` focused on project-specific overlays, constraints, and checks

### 4. `docs/business-plan.md` — Business Plan

**Location:** `docs/`
**Audience:** Founders, investors, stakeholders
**Purpose:** All business context — revenue, GTM, competitors, regulatory, team, funding

**Content includes:**

- Executive summary
- Problem/solution (narrative form, not technical)
- Market analysis (TAM/SAM/SOM)
- Competitive landscape
- Revenue model + financial projections
- Go-to-market strategy
- Regulatory path
- Risk analysis
- Team + hiring plan
- Funding strategy

**Rules:**

- No code, no schemas, no API endpoints — purely business content
- Technical architecture can be summarized at a high level but not detailed
- This is the only doc that should contain revenue projections, competitor tables, and GTM phases

**Note:** New projects should use `docs/business-plan.md`. Existing projects may use other names (e.g., `docs/pricing.md` in ChickenDuck) — no retroactive rename required.

---

## Constraint Documentation

Constraints can be documented in two ways depending on project complexity:

### Simple (Genaurasity, Imago)

- Constraints live directly in the manifest §3 as a numbered table
- CLAUDE.md has a trigger table pointing to manifest section numbers
- AGENTS.md has a summary table matching the same numbering

### Complex (ChickenDuck)

- Each constraint gets its own file in `.claude/docs/01-*.md`, `02-*.md`, etc.
- `.claude/docs/README.md` indexes all constraint files
- CLAUDE.md trigger table points to individual constraint docs
- Manifest §3 contains the summary; detail lives in the constraint files

**When to split:** If a single constraint needs more than ~10 lines of explanation (edge cases, code examples, state machines), give it its own file.

---

## Sync Guidelines

When making changes, keep these docs in sync:

| Change Type           | Update                                                             |
| --------------------- | ------------------------------------------------------------------ |
| New constraint        | Manifest §3 + CLAUDE.md trigger table + AGENTS.md constraint table |
| New API endpoint      | Manifest §6                                                        |
| Schema change         | Manifest §7 (DDL) + migration file                                 |
| New key file          | Manifest §8                                                        |
| New reference pattern | CLAUDE.md common patterns table                                    |
| Stack change          | Manifest §2 + CLAUDE.md "at a glance"                              |
| Business model change | `docs/business-plan.md` only (never manifest)                      |
| New dev command       | CLAUDE.md development workflow                                     |
| New known issue       | Manifest §9                                                        |

**General rule:** If you're unsure where something belongs, ask: "Is this a technical fact or a business fact?" Technical → manifest. Business → business plan. Quick reference → CLAUDE.md.

---

## Structural Parity Across Projects

All three projects should have the same section numbers and headings in their manifests and CLAUDE.md files. This allows agents to transfer knowledge between projects:

**Manifest sections:** 1-Overview, 2-Stack, 3-Constraints, 4-Structure, 5-Domain Logic, 6-API, 7-Schema, 8-Key Files, 9-Known Issues, 10-Resources

**CLAUDE.md sections:** Header pointers, At a Glance (with trigger table), Project-Specific Conventions (overrides only), Common Patterns, Development Workflow, Multi-Agent Workflow

Section content varies by project, but the structure stays consistent.

---

## Global vs Project-Level Conventions

Shared coding conventions (TypeScript, React, Python, file naming, DRY/SOLID, web research tools) live in the global `~/.claude/CLAUDE.md` and apply to all projects automatically. Project-level CLAUDE.md files should only include:

- Conventions that **override** the global defaults (e.g., different file naming scheme)
- Conventions that are **unique** to the project (e.g., Zustand stores, design tokens, specific linting config)
- A note pointing to the global file: `> General coding conventions are in the global ~/.claude/CLAUDE.md`
