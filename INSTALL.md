# Installation

End-to-end setup for the dual-model hybrid workflow. Plan for ~30 minutes
the first time. Subsequent projects only need step 4.

## Prerequisites

- **Claude Code** installed and working — https://claude.com/claude-code
- **Codex** (or another GPT-class model with file-system access via a
  plugin/CLI) installed and signed in
- **Git** + **GitHub CLI (`gh`)** if you want issues created automatically
  by `/scope` (optional — local-only fallback works if `gh` isn't available)

## 1. Drop the kit into your global Claude Code config

```bash
git clone https://github.com/{your-handle}/claude-codex-hybrid-kit.git
cd claude-codex-hybrid-kit

# Copy slash commands, agent files, and the master workflow doc
cp -r .claude/commands ~/.claude/
cp -r .claude/agents   ~/.claude/
cp    .claude/WORKFLOW.md ~/.claude/
```

If you already have a `~/.claude/commands/` directory, the `cp -r` above
will merge — your existing commands stay, kit commands land alongside.
Inspect the diff before overwriting anything you've customized.

Verify Claude Code picks them up: open Claude Code in any project and
type `/`. You should see `/scope`, `/build`, `/verify`, `/implement`,
`/implement-phase`, etc. in the command list.

## 2. Place the shared docs

The kit's `shared-docs/GPT_ARCHITECT_REVIEW_STANDARD.md` and
`shared-docs/PROJECT_DOCUMENTATION_STANDARD.md` are designed to sit
**at the parent of your projects** so multiple projects under one
parent directory share them.

The recommended layout:

```
~/code/                                    # or wherever your projects live
├── docs/                                  # ← shared-docs go here
│   ├── GPT_ARCHITECT_REVIEW_STANDARD.md
│   └── PROJECT_DOCUMENTATION_STANDARD.md
├── project-one/
│   ├── AGENTS.md          # references ../docs/GPT_ARCHITECT_REVIEW_STANDARD.md
│   ├── CLAUDE.md
│   └── docs/
│       └── context/...
└── project-two/
    └── ...
```

```bash
# Run from inside the kit clone
mkdir -p ~/code/docs
cp shared-docs/*.md ~/code/docs/
```

If your projects don't share a common parent, you can drop the shared
docs anywhere reachable and just adjust the relative path in each
project's `AGENTS.md` (the "Shared Standard" section).

### Why this layout

Pre-2026 versions of this workflow embedded the full `/architect` and
`/review` contract inline in every project's `AGENTS.md` (~380 lines per
project). The current version extracts that into one canonical
`GPT_ARCHITECT_REVIEW_STANDARD.md`. Project AGENTS.md files become small
overlays (~120 lines) that only declare project-specific context and
checks.

For token efficiency, Codex does **partial reads** of the shared
standard:

- For `/architect`, it reads only `## /architect` + `## Handover Contract`
- For `/review`, it reads only `## /review` + `## Handover Contract`

That keeps token usage close to the old single-file project guides while
preserving one shared contract. The standard declares **stable signpost
headers** (`## /architect`, `## /review`, `## Handover Contract`,
`### Critical Issues (🔴)`, `### Important Issues (🟡)`,
`### Minor Issues (🟢)`) that the partial-read pattern depends on. If you
fork the standard, don't rename those headers — adding/removing lines
inside sections is fine.

## 3. Confirm Codex picks up project AGENTS.md

Codex reads `AGENTS.md` from the project root automatically when you run
slash commands inside that project. There's no global Codex install step
for this kit — the project AGENTS.md (which you'll create in step 4) is
the entry point.

If you're using a different GPT-class plugin/CLI, check its docs for how
it loads project-level instructions. The pattern is: read `AGENTS.md`,
follow the relative-path pointer to `GPT_ARCHITECT_REVIEW_STANDARD.md`,
do partial reads of that standard.

## 4. Install templates into a project

For each project where you want to use this workflow:

```bash
cd <project-root>

# Pull templates from the kit clone
KIT=~/path/to/claude-codex-hybrid-kit

cp $KIT/templates/CLAUDE.md.template       CLAUDE.md
cp $KIT/templates/AGENTS.md.template       AGENTS.md

mkdir -p docs/context
cp $KIT/templates/docs/context/ProjectName.md.template  docs/context/<ProjectName>.md
cp $KIT/templates/docs/business-plan.md.template        docs/business-plan.md

mkdir -p .claude/implementations/archive

# Optional: per-constraint detail docs (only if a constraint needs more
# than ~10 lines of explanation — otherwise leave it in the manifest §3)
mkdir -p .claude/docs
cp $KIT/templates/constraint-doc.md.template  .claude/docs/01-{your-constraint-slug}.md
```

Then **fill in every `{placeholder}`** in:

- `CLAUDE.md` — stack, dev commands, conventions, common patterns
- `AGENTS.md` — project context pointers, key facts, constraints summary,
  project-specific architect/review overlays. Keep the "Shared Standard"
  section's relative path correct for your layout.
- `docs/context/<ProjectName>.md` — full technical manifest (10 sections)
- `docs/business-plan.md` — business context, if applicable

The manifest is the heaviest one; expect 30–60 minutes the first time.
Once it's filled in, every subsequent issue benefits from it.

## 5. Run your first hybrid issue

```
# In Claude Code, inside your project:
/scope "Add a /health endpoint that returns {status, version, uptime}"

# (Claude asks alignment questions, explores the codebase, creates
# .claude/implementations/issue-N.md and a GitHub issue if gh is set up)

# Switch to Codex (in the same project):
/architect 1

# (Codex appends Architecture Plan - Attempt 1 to the issue file)

# Back in Claude Code:
/build 1

# (Claude implements, writes tests, appends Implementation Notes)

# Codex:
/review 1

# (Codex appends GPT Review - Attempt 1, ending with APPROVE / REQUEST
#  CHANGES / BLOCK. If anything other than APPROVE, loop /build → /review.)

# Once APPROVED, Claude Code:
/verify 1

# (Claude addresses any remaining review issues, runs tests, generates
#  the manual test plan, waits for your sign-off, commits.)
```

## Troubleshooting

### Slash commands don't show up in Claude Code

Check `ls ~/.claude/commands/`. If the files aren't there, `cp` failed
silently. Re-run step 1.

If they're there but Claude Code doesn't list them: restart the Claude
Code session (open a new chat in the same project).

### Codex says it can't find GPT_ARCHITECT_REVIEW_STANDARD.md

Check the relative path in your project's `AGENTS.md` "Shared Standard"
section. The default in the template is `../docs/...` which assumes the
shared docs sit at the parent of your project. Adjust if your layout
differs.

### Issue file isn't being appended to (sections overwrite each other)

That violates the append-only contract. Common cause: Codex inserted a
review block in the middle of the file instead of at EOF. The standard
specifies that `/review` must verify the write with `tail -n 20` after
appending — if your Codex setup doesn't do that, add it manually for now,
or open an issue on the kit.

### Tests pass on Claude's side but fail in CI

The kit ships with placeholder `{test_command}` / `{typecheck_command}` /
`{lint_command}` references that resolve to whatever you declare in your
project's `CLAUDE.md`. If those declarations are wrong (e.g. missing the
`cd backend &&` prefix in a monorepo), Claude runs the wrong commands.
Fix the project's `CLAUDE.md` Development Workflow section.

### Codex burns way more tokens than expected

Confirm Codex is doing partial reads of the shared standard. The
"Efficient Read Pattern" section of `GPT_ARCHITECT_REVIEW_STANDARD.md`
documents this. If your Codex setup loads the whole file every time,
the partial-read pattern isn't being honored. Check your project
AGENTS.md "Shared Standard" section — it should explicitly list which
sections to load for `/architect` vs `/review`.

## Updating the kit

```bash
cd ~/path/to/claude-codex-hybrid-kit
git pull

# Re-copy the bits that may have changed
cp -r .claude/commands  ~/.claude/
cp -r .claude/agents    ~/.claude/
cp    .claude/WORKFLOW.md ~/.claude/
cp    shared-docs/*.md  ~/code/docs/
```

Templates intentionally **don't** auto-update — once you've customized
your project files, you own them. Diff against the kit's templates
manually if you want to pull in upstream improvements.
