# BMAD Synthesize — Documentation Generator

## Purpose

Read `_bmad-output/` artifacts and produce standardized project documentation following `PROJECT_DOCUMENTATION_STANDARD.md`.

## Prerequisites

- `_bmad-output/product-brief.md` exists
- `_bmad-output/prd.md` exists
- `_bmad-output/architecture.md` exists

## Execution Instructions

When the user says "/bmad-synthesize", execute these steps:

### Step 1: Read All BMAD Artifacts

```
READ: _bmad-output/product-brief.md
READ: _bmad-output/prd.md
READ: _bmad-output/architecture.md
READ: PROJECT_DOCUMENTATION_STANDARD.md (the canonical spec — see this kit's
      `shared-docs/PROJECT_DOCUMENTATION_STANDARD.md`, or wherever your fork
      placed the shared docs; commonly at the parent of your projects, e.g.
      `../docs/PROJECT_DOCUMENTATION_STANDARD.md`)

IF any file is missing:
  OUTPUT: "Missing BMAD artifact: {filename}
  Run /bmad-start first to generate all 3 artifacts."
  STOP
```

### Step 2: Generate CLAUDE.md

Follow PROJECT_DOCUMENTATION_STANDARD §1 (CLAUDE.md — Quick Reference).

**Source Mapping:**

| Section              | Source                                               |
| -------------------- | ---------------------------------------------------- |
| At a Glance          | architecture.md tech stack + prd.md summary          |
| Critical Constraints | prd.md constraints → trigger table format            |
| Coding Conventions   | architecture.md stack → derive conventions           |
| Common Patterns      | Leave as TBD (populated after first implementations) |
| Development Workflow | architecture.md hosting + stack → setup commands     |
| Multi-Agent Workflow | Standard 5-phase template from existing projects     |

**Rules:** Max 150 lines. No business logic. Trigger table only.

```
OUTPUT: Write CLAUDE.md to project root.
```

### Step 3: Generate Manifest

Follow PROJECT_DOCUMENTATION_STANDARD §2 (Technical Manifest — 10 sections).

**Source Mapping:**

| Section          | Source                                               |
| ---------------- | ---------------------------------------------------- |
| §1 Overview      | product-brief.md (problem, solution, users)          |
| §2 Tech Stack    | architecture.md (stack table)                        |
| §3 Constraints   | prd.md (constraint table with rule + impact)         |
| §4 Structure     | architecture.md (folder structure)                   |
| §5 Domain Logic  | prd.md (epics, stories, business rules)              |
| §6 API Endpoints | architecture.md (API design)                         |
| §7 Schema        | architecture.md (data model → full DDL)              |
| §8 Key Files     | Leave as TBD (populated after first implementations) |
| §9 Known Issues  | architecture.md (readiness blockers)                 |
| §10 Resources    | Collect external links from all artifacts            |

**Rules:** Max 500 lines. No business content. Full DDL in §7.

```
OUTPUT: Write docs/context/{ProjectName}.md
```

### Step 4: Generate AGENTS.md

Follow PROJECT_DOCUMENTATION_STANDARD §3 (AGENTS.md).

**Source Mapping:**

- Role definition: Standard template (GPT = Architect + Critic, Claude = Scope + Build + Verify)
- Constraint table: Copy from manifest §3 (same numbering)
- Output templates: Standard /architect and /review formats

**Quality Enhancement Awareness (include in generated AGENTS.md):**

- REQ-XX traceability in architecture template: If issue has REQ-XX tags, architect references them in implementation steps and test strategy
- STRIDE quick assessment table in security analysis section of review template
- AI bias observations section (acceleration, scope, over-optimization) in review template
- REQ-XX traceability verification table in review template (if tags present)

```
OUTPUT: Write AGENTS.md to project root.
```

### Step 5: Generate Business Plan

Follow PROJECT_DOCUMENTATION_STANDARD §4 (Business Plan).

**Source Mapping:**

- Executive summary: product-brief.md
- Problem/Solution: product-brief.md (narrative form)
- Market analysis: product-brief.md (market, users, competitors)
- Revenue model: prd.md + product-brief.md (pricing, monetization)
- Regulatory: prd.md (constraints with regulatory implications)

```
OUTPUT: Write docs/business-plan.md
```

### Step 6: Create GitHub Repository and Issues

```
ACTION: Create repo (if not exists)
EXECUTE: gh repo create {project-name} --private

ACTION: Create milestones from epics in prd.md
FOR EACH epic:
  EXECUTE: gh api repos/{owner}/{project-name}/milestones -f title="{epic_name}" -f description="{epic_description}"

ACTION: Create issues from user stories (1 story = 1 issue, assigned to milestone)
FOR EACH user_story:
  EXECUTE: gh issue create --title "{story_title}" --body "{story_body}" --milestone "{epic_name}"

NOTE: Do NOT pre-create issue-{N}.md files. These are created on-demand
when you start working on each issue via /scope #N or /implement #N —
matching the existing workflow pattern. Only the directory is created here.
```

### Step 7: Create Supporting Structure

```
ACTION: mkdir -p .claude/implementations/
ACTION: mkdir -p .claude/docs/ (if project has complex constraints)
ACTION: mv _bmad-output/ _bmad-output-archive/
```

### Step 8: Report Completion

```
OUTPUT:
"DOCUMENTATION SYNTHESIS COMPLETE

Created:
✅ CLAUDE.md ({line_count} lines)
✅ docs/context/{ProjectName}.md ({line_count} lines)
✅ AGENTS.md
✅ docs/business-plan.md
✅ GitHub repo: {repo_url}
✅ {milestone_count} milestones, {issue_count} issues created
✅ .claude/implementations/ directory ready

BMAD artifacts archived to: _bmad-output-archive/

Your project is ready for development.
Pick an issue and start:
- Hybrid workflow: /scope #{first_issue} (then /architect, /build, /review, /verify)
- Single-model: /implement #{first_issue}
- Phase mode: /implement-phase #{first_issue}

Local issue-{N}.md files will be created on-demand when you start each issue."
```
