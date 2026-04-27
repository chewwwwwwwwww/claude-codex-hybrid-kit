# Scope Command (Hybrid Phase 1)

## Path Resolution (IMPORTANT)

This command operates on the CURRENT PROJECT, not global .claude/.

When reading/writing files:

- `.claude/implementations/issue-123.md` = `{project}/.claude/implementations/issue-123.md`
- NOT `~/.claude/implementations/issue-123.md`

## GitHub Repository Resolution (CRITICAL)

**Before any GitHub operations, determine the correct repository:**

1. **Check CLAUDE.md** (in project root):
   - Look for line starting with `**Repository:**`
   - This is the PREFERRED method - project explicitly declares its repo

2. **Fall back to git remote**:
   - Run: `git remote get-url origin`
   - Parse: `git@github.com:owner/repo.git` -> `owner/repo`
   - Or: `https://github.com/owner/repo.git` -> `owner/repo`

3. **Store as {github_repo}** for all GitHub operations

**NEVER hardcode a repository name. Always resolve dynamically.**

## Purpose

Hybrid Phase 1: Context & Scoping. Explores the codebase, identifies relevant files and constraints, and creates the issue file for GPT to architect against.

**What this command does:**

- Creates GitHub issue
- Explores codebase for relevant files, patterns, and code
- Reads applicable constraint docs
- Creates issue-{N}.md with `Mode: Hybrid` header
- Fills in `## Codebase Exploration` section

**What this command does NOT do:**

- Write code
- Create architecture plans (that's GPT's job)
- Run tests

---

## Execution Instructions

When the user says "/scope", "/scope #N", or "Scope this feature", execute these steps:

### Requirement Alignment (Standing Directive)

Throughout every step below, use the `AskUserQuestion` tool **extensively** whenever
requirements are ambiguous, under-specified, or have meaningful trade-offs.

- Ask up-front (Step 1) if the raw requirements leave key behaviors undefined
- Ask during exploration (Step 4) if the codebase reveals multiple reasonable paths
- Before writing Step 6, triage every open question into two buckets:
  - **Ask the user now** — product intent, UX preferences, scope boundaries,
    acceptance criteria, priorities, and anything the user can answer from
    domain knowledge alone
  - **Defer to the architect** — implementation trade-offs, structural design
    choices, and technical decisions that require reading surrounding code to
    answer well

Prefer asking the user over making assumptions on product/scope questions. It is
cheaper to clarify at scope time than to rework at build time. Questions that
survive this triage still go into the "Questions for Architect" section of the
issue file — that section should not be empty just because you asked the user
some questions.

### Step 1: Parse Input

```
IF user message contains issue number (#N):
  {issue_number} = N
  ACTION: Fetch existing GitHub issue from {github_repo}
  STORE: {requirements} = issue body
  PROCEED to Step 3

ELSE IF user message contains requirements or references a plan file:
  EXTRACT: Requirements from message or file
  STORE: {requirements}
  PROCEED to Step 2

ELSE:
  OUTPUT: "I need requirements to scope. Either:
  1. Share the feature requirements
  2. Reference a plan file: '/scope - use plan from .claude/plan-X.md'
  3. Reference an existing issue: '/scope #123'"
  STOP
```

### Step 2: Create GitHub Issue

```
ACTION: Create new GitHub issue in {github_repo}

EXTRACT TITLE:
- From first header or opening lines of requirements
- Generate concise title if not found (max 80 chars)

GITHUB ISSUE STRUCTURE:
Repository: {github_repo}
Title: {title}
Body: {requirements}
Labels: ["hybrid-workflow"] (optional)

EXECUTE: Create issue
RESULT: {issue_number}

ERROR HANDLING:
IF GitHub creation fails:
  OUTPUT: "Could not create GitHub issue. Continue with local tracking? (yes/no)"
  IF "yes":
    {issue_number} = "local-{timestamp}"
    {github_enabled} = false
  ELSE:
    STOP

IF successful:
  {github_enabled} = true
  {issue_url} = "https://github.com/{github_repo}/issues/{issue_number}"
```

### Step 3: Read Project Manifest

```
ACTION: Read the project manifest, located at `docs/context/{ProjectName}.md`
        per `PROJECT_DOCUMENTATION_STANDARD.md`. Also read `CLAUDE.md` in the
        project root for the constraint trigger table and conventions.
PURPOSE: Understand project structure, tech stack, domain logic
STORE: Key context for exploration
```

### Step 4: Explore Codebase

```
ACTION: Systematically explore the codebase to find files relevant to the requirements

FOR EACH requirement/feature area:

  4a. Find relevant files:
  - Use Glob to find files by name patterns
  - Use Grep to search for keywords, function names, API endpoints
  - Look for existing implementations of similar features
  - Check the project's data layer (database schema, ORM models, type definitions)
    if the feature involves persistent data

  4b. Identify existing patterns:
  - How are similar features structured?
  - What hooks, components, or services are reused?
  - What's the data flow for this feature area?

  4c. Read key code sections:
  - Read relevant source files to understand interfaces and contracts
  - Note function signatures, type definitions, API shapes
  - Identify extension points where new code should hook in

  4d. Check database / data-layer schema:
  - If the feature involves data, read migration files, ORM models, or use the
    project's documented schema-introspection MCP/CLI to list tables
  - Note relevant table structures, relationships, and access policies

  4e. Security-relevant exploration:
  - If the feature touches auth, payments, AI, or user data:
    - Note existing access policies on affected tables/collections
    - Check where sensitive fields (subscription_status, rate_limits, credits,
      role, is_admin) are stored, and whether the user can write them directly
    - Note existing rate-limiting patterns in the codebase
    - Identify any environment variables the feature will need (and whether
      they belong on the client or server)
    - Flag if new API endpoints will be created (GPT should design with auth
      + rate limiting from the start)

COLLECT: File paths, line ranges, code snippets, patterns found
```

### Step 5: Read Applicable Constraints

```
ACTION: Read .claude/docs/README.md for the constraint index

FOR EACH constraint doc:
  CHECK: Does this feature touch the constraint's area?
  (Use the "When to Read" keywords from README.md)

  IF relevant:
    READ: The full constraint doc
    EXTRACT: Key rules that apply to this feature
    STORE: Constraint summaries

RESULT: List of applicable constraints with summaries
```

### Step 5.5: Assign Requirement Traceability Tags (Complex Features)

```
ACTION: Count acceptance criteria in {requirements}

IF acceptance criteria count >= 5:
  ASSIGN REQ-XX tags:
  - Number each acceptance criterion: REQ-01, REQ-02, REQ-03, etc.
  - Prepend tag to each criterion in {requirements}
  - STORE: {traceability} = "REQ-XX tags assigned (complex feature)"

  NOTE: Tags will be referenced downstream:
  - GPT /architect: "Component X implements REQ-01, REQ-03"
  - Claude /build: "REQ-02 implemented in {file}:{function}"
  - Testing: "REQ-01 ✅ verified by test {name}"

ELSE:
  STORE: {traceability} = "Simple feature (< 5 criteria) — no REQ tags"

Tags live ONLY in the issue markdown file. Not in filenames, code comments, or git messages.
```

### Step 6: Create Issue File

```
ACTION: mkdir -p .claude/implementations/
ACTION: Create .claude/implementations/issue-{issue_number}.md

CONTENT:
---
# Issue #{issue_number}: {title}

**Status:** Scoping
**Mode:** Hybrid
**Created:** {date}
**GitHub:** {issue_url or "Local only"}
**Repository:** {github_repo}
**Traceability:** {traceability}

## Original Requirements

{If REQ-XX assigned:}
**Requirements (REQ-XX tagged for traceability):**

- REQ-01: {criterion 1}
- REQ-02: {criterion 2}
- REQ-03: {criterion 3}
...

{If not assigned:}
{requirements}

---

## Codebase Exploration (Claude /scope)

**Agent:** Claude
**Date:** {date}

### Relevant Files

| File | Purpose | Lines |
| ---- | ------- | ----- |
{For each relevant file found in Step 4}

### Existing Patterns

{How similar features are implemented in this codebase.
Reference specific files and describe the pattern.
GPT will read these files directly.}

### Key Code Snippets

{Include actual code blocks with file path + line range annotations.
Focus on interfaces, type definitions, and extension points
that the architect needs to see.
Keep to the most important 5-10 snippets.}

### Applicable Constraints

{For each relevant constraint from Step 5:}
- **{constraint number}: {name}** - {summary of how it applies to this feature}
  - Key rules: {bullet points}
  - Read: `.claude/docs/{filename}` for full details

### Database Schema (if relevant)

{Table structures, relationships, RLS policies that matter}

### Security Considerations (if applicable)

{Only include if the feature touches auth, payments, AI, user data, or new API endpoints.
This helps the architect design with security in mind from the start.}

- **RLS implications:** {Do new/modified tables need RLS policies? Are sensitive fields involved?}
- **Sensitive operations:** {Will this feature call AI providers, payment processors, or email services? These must go through the backend.}
- **Rate limiting:** {Does this feature need rate limiting? What existing patterns does the codebase use?}
- **Environment variables:** {What new secrets will this feature need? Which are safe for client vs server-only?}

### Questions for Architect

{Ambiguities in the requirements that need architectural decisions.
Things Claude noticed during exploration that GPT should consider.
Trade-offs that the architect should weigh.

NOTE: User-answerable questions (product intent, UX, scope, priorities) should
already have been resolved via `AskUserQuestion` per the Standing Directive.
Items listed here should be genuinely architectural — decisions that require
reading surrounding code or weighing technical trade-offs that the architect
is better positioned to make.}

---
*Next step: Open GPT (Codex) and run `/architect {issue_number}` to create the architecture plan.*
---
```

### Step 7: Report Completion

```
OUTPUT:
"---
HYBRID PHASE 1: SCOPE COMPLETE
---

Issue: #{issue_number} - {title}
{if github_enabled: "GitHub: " + issue_url}
File: .claude/implementations/issue-{issue_number}.md

Explored:
- {count} relevant files identified
- {count} existing patterns documented
- {count} constraints applicable
- {count} questions for architect

Next Step:
Open GPT (Codex) and run:
  /architect {issue_number}

GPT will read the issue file, explore the referenced code,
and write the Architecture Plan section.
---"
```

---

## What This Command Does NOT Do

- Write any code
- Create an architecture plan (GPT does this in Phase 2)
- Run tests or linting
- Make commits
- Modify existing source files

## What It Produces

- GitHub issue (if GitHub accessible)
- `.claude/implementations/issue-{N}.md` with Mode: Hybrid
- Codebase Exploration section directing GPT to the right files
