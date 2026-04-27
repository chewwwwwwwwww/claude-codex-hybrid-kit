# GPT /architect & /review Standard

Canonical cross-project contract for GPT/Codex in the hybrid Claude + GPT workflow across `/VS`.

This file owns the universal behavior for `/architect` and `/review`.
Its default shape follows the mature ChickenDuck hybrid workflow: spelled-out command behavior, stable section headings, and explicit review decision semantics.
Each project `AGENTS.md` should own only:

- workflow ownership
- project context files to read
- key facts and critical constraints summary
- project-specific architecture emphasis
- project-specific review checklist
- any explicit deviation from this standard

## Expected Project Overlay

Each project `AGENTS.md` should provide:

- a pointer to this file
- project context pointers (`docs/context/...`, `CLAUDE.md`, extra constraint docs)
- critical constraints summary
- `/architect` overlays:
  - extra context files to read
  - architecture-specific design emphases
- `/review` overlays:
  - extra verification docs to read
  - project-specific checklist heading and items
- handover deviations, if any

If a project does not explicitly deviate, this file is the source of truth.

## Efficient Read Pattern

Do not load this entire file unless you need to.

For `/architect`, read:

- the project `AGENTS.md`
- this file's `## /architect` section
- this file's `## Handover Contract` section

For `/review`, read:

- the project `AGENTS.md`
- this file's `## /review` section
- this file's `## Handover Contract` section

That keeps token usage closer to the old single-file project guides while preserving one shared contract.

### Header Stability

The efficient read pattern is based on section headers, not fixed line numbers.

- Adding or removing lines inside a section is fine.
- Reordering unrelated paragraphs inside a section is fine.
- Renaming or removing the main section headers is a breaking change for the read pattern.

Treat these headers as stable signposts:

- `## /architect`
- `## /review`
- `## Handover Contract`
- `### Critical Issues (🔴)`
- `### Important Issues (🟡)`
- `### Minor Issues (🟢)`

## Shared Rules

- Read the project `AGENTS.md` first. It tells you which project docs and overlays apply.
- The coordination file is `.claude/implementations/issue-{N}.md`.
- Treat the issue file as append-only. Never overwrite earlier sections.
- Write your output into the issue file, not just into chat.
- Use `file:line` references for concrete findings and implementation guidance.
- If the issue uses `REQ-XX` tags, propagate them through architecture, review, and test recommendations.
- Prefer stable, repeatable section headings. If a section does not apply, keep the heading and write `None.` rather than inventing a new structure.

### Attempt Numbering

For `/architect`:

- If `## Architecture Plan - Attempt {N}` sections exist, increment the highest attempt number.
- If only a legacy unnumbered architecture section exists, treat it as Attempt 1 and write the next numbered section as Attempt 2.

For `/review`:

- If `## GPT Review - Attempt {N}` sections exist, increment the highest attempt number.
- If only a legacy unnumbered GPT review section exists, treat it as Attempt 1 and write the next numbered section as Attempt 2.

## /architect

### Goal

Produce a buildable, scoped implementation plan that gives Claude enough specificity to implement correctly without inventing missing architecture.

### Read Order

1. Read the project `AGENTS.md`.
2. Read `.claude/implementations/issue-{N}.md`.
3. Determine the architecture attempt number.
4. If this is Attempt 2+, read previous architecture attempts and the review or feedback that triggered the retry.
5. Read the project context files named in `AGENTS.md`.
6. Read the source files listed in the issue's `Codebase Exploration`.
7. Explore adjacent code on your own to validate scope, patterns, and risks.

### Planning Rules

- Treat the issue file as the scoped source of truth for this issue.
- Favor existing files and patterns over introducing parallel systems.
- Keep the blast radius narrow. Do not solve unrelated cleanup unless it is required to make the feature correct.
- Be specific enough that implementation can proceed without another design round:
  - name files
  - name functions, routes, tables, fields, and state holders
  - call out migrations, DTOs, and tests explicitly
- Design for testability. Every meaningful branch should have an obvious test target.
- Surface true ambiguity in `Open Questions`; do not hide missing decisions behind vague prose.

### Retry Preamble

If this is Attempt 2+, start with:

```markdown
### Changes from Attempt {N-1}

- **What changed:** {Summary of architectural changes}
- **Issues addressed:** {List issues from previous review or feedback that prompted the retry}
- **What's preserved:** {Parts of the previous plan that remain unchanged}
```

### Output Template

```markdown
## Architecture Plan - Attempt {N}

**Agent:** GPT
**Date:** {date}

### Approach

{2-3 paragraphs describing the overall approach, core design decisions, and rationale}

### Implementation Steps

1. **{file path}** - {Action: Create/Modify/Delete} {(implements REQ-XX, if present)}
   - {What to do and why}
   - {Specific details: function signatures, data structures, constraints, migration notes}

2. **{file path}** - {Action} {(implements REQ-XX, if present)}
   - {Details}

{Continue for all files in implementation order}

### Data Model Changes

{Tables, columns, indexes, migrations, RLS, triggers, or `None.`}

### API Changes

| Endpoint | Method | Request | Response | Notes |
| -------- | ------ | ------- | -------- | ----- |
| {path} | {verb} | {request} | {response} | {notes} |

### Frontend Changes

{Component hierarchy, state management, navigation, and user flow, or `None.`}

### Test Strategy

1. `test_{scenario_name}` - {What it verifies} {(verifies REQ-XX, if present)}
2. `test_{scenario_name}` - {What it verifies} {(verifies REQ-XX, if present)}

### Risk Assessment

- **{Risk}:** {Mitigation}

### Constraint Compliance

{Relevant constraints and how the design stays compliant}

### Open Questions

{Open questions that truly need resolution, or `None.`}

### Recommendation

**{READY TO BUILD | NEEDS CLARIFICATION | REQUIREMENTS UNCLEAR}**

{Brief justification}
```

## /review

### Goal

Perform a scoped critic and security review of Claude's implementation for the current attempt, focusing on correctness, regressions, compliance, and missing tests.

### Scope Rule

Only review files listed in the issue's `Implementation Notes`.
Ignore unrelated working tree changes entirely.
Do not flag unrelated diffs as scope creep.

### Read Order

1. Read the project `AGENTS.md`.
2. Read `.claude/implementations/issue-{N}.md`.
3. Determine the review attempt number.
4. If this is Attempt 2+, read previous GPT review attempts and the current attempt's `Implementation Notes`.
5. Read the changed files listed in `Implementation Notes`.
6. Run `git diff -- <file1> <file2> ...` using only those files. Never run a bare `git diff`.
7. Read the project context and constraint docs named in `AGENTS.md`.

### Review Standards

- Findings come first. Summary is secondary.
- Stay scoped to correctness, regressions, compliance, security, and missing verification.
- Use concrete evidence from the changed files and diff.
- Recommend specific fixes when possible.
- Keep severity calibrated:
  - `Critical`: merge-blocking correctness, security, compliance, or requirement failure
  - `Important`: should be fixed before handoff completion, but not an immediate catastrophic blocker
  - `Minor`: acceptable to defer

### Retry Verification

If this is Attempt 2+, start with:

```markdown
### Previous Issues Verification

| Issue | Previous Severity | Status |
| ----- | ----------------- | ------ |
| {issue description} | 🔴 Critical / 🟡 Important / 🟢 Minor | FIXED / STILL PRESENT / PARTIALLY FIXED |
```

### Output Template

```markdown
## GPT Review - Attempt {N}

**Agent:** GPT
**Date:** {date}

### Overall Assessment

{1-2 paragraphs summarizing implementation quality and completeness}

### Critical Issues (🔴)

- **[{file}:{line}]** {Problem description}
  - **Impact:** {What breaks or is at risk}
  - **Fix:** {Specific fix}

{Or `None.`}

### Important Issues (🟡)

- **[{file}:{line}]** {Problem description}
  - **Impact:** {What degrades}
  - **Fix:** {Specific fix or direction}

{Or `None.`}

### Minor Issues (🟢)

- **[{file}:{line}]** {Suggestion} -- {Rationale}

{Or `None.`}

### Security Analysis

#### Threat Areas Affected

{Auth, payments, PII, RAG, data access, task execution, or `None.`}

#### OWASP Quick Check

- [ ] Injection (SQL, XSS, command, prompt injection if relevant)
- [ ] Broken Authentication
- [ ] Sensitive Data Exposure
- [ ] Broken Access Control
- [ ] Security Misconfiguration

#### STRIDE Quick Assessment

| Category | Applicable? | Finding |
| -------- | ----------- | ------- |
| Spoofing | [Yes/No/N/A] | {finding} |
| Tampering | [Yes/No/N/A] | {finding} |
| Repudiation | [Yes/No/N/A] | {finding} |
| Information Disclosure | [Yes/No/N/A] | {finding} |
| Denial of Service | [Yes/No/N/A] | {finding} |
| Elevation of Privilege | [Yes/No/N/A] | {finding} |

#### {Project}-Specific Checks

{Reuse the checklist heading and items defined in the project `AGENTS.md`, marking each item}

### Additional Tests Recommended

1. `test_{scenario}` - {What it should verify and why}

{Or `None.`}

### Frontend Flow Tests Recommended

1. {Flow name} - {URL} - {Account type}
   - Steps: {Interactions}
   - Expected: {Outcome}

{Or `None.`}

### Positive Highlights

{What was done well, or `None.`}

### AI Bias Observations

- **Acceleration:** {Observation}
- **Scope:** {Observation}
- **Over-optimization:** {Observation}

### Requirement Traceability Verification

| REQ Tag | Requirement | Implemented? | Tested? |
| ------- | ----------- | ------------ | ------- |
| REQ-01 | {criterion} | [Yes/No/Partial] | [Yes/No] |

{If no REQ-XX tags exist, replace this section body with `None.`}

### Recommendation

**{APPROVE / REQUEST CHANGES / BLOCK}**

{Brief justification}
```

### Decision Guidelines

Do not include these lines in the output.

- `BLOCK` should only be used for issues in the actual changed files for this issue.
- Do not block for unrelated working tree changes. Those are outside the review scope.
- `APPROVE` means the user can hand the issue back to Claude for `/verify #N`.
- `REQUEST CHANGES` means the user can hand the issue back to Claude for `/verify #N`, with Claude addressing the scoped review issues.
- `BLOCK` means the user should return to `/build #N` for an implementation retry, or `/architect #N` if the design itself must change.

### Recommendation Semantics

- `APPROVE`: no remaining critical or important issues in the scoped changed files for this issue
- `REQUEST CHANGES`: at least one important issue remains, but no critical blocker
- `BLOCK`: at least one critical correctness, security, compliance, or requirement gap remains in the scoped changed files for this issue, or the issue needs redesign rather than another small implementation pass

## Handover Contract

The issue file is the handoff artifact between Claude and GPT.

### Section Ownership

| Section | Written By | When |
| ------- | ---------- | ---- |
| Original Requirements | Claude (`/scope`) | Phase 1 |
| Codebase Exploration | Claude (`/scope`) | Phase 1 |
| Architecture Plan - Attempt {N} | GPT (`/architect`) | Phase 2 |
| Implementation Notes | Claude (`/build`) | Phase 3 |
| GPT Review - Attempt {N} | GPT (`/review`) | Phase 4 |
| Verification & Commit | Claude (`/verify`) | Phase 5 |

### Write Rules

- Append new sections. Never edit old ones in place.
- For `/review`, append the new review block to the physical end of `.claude/implementations/issue-{N}.md`.
- Do not insert the block after a matching heading, marker, or earlier review section.
- After writing a review block, verify the write with `tail -n 20` plus line-numbered output to confirm the new block is at EOF.
- If the block is not at EOF, fix it before responding.
