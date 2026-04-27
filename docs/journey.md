# The 5-Month Journey to a Dual-Model Workflow

A first-person account of how this kit's setup evolved between December
2025 and April 2026. Read this if you want context for _why_ the workflow
looks the way it does. Skip it if you just want to use the kit — the
README has everything you need to ship.

---

## Stage 0: Native plan mode + dangerously bypass

When I started using Claude Code, I used the native plan mode straight
out of the box. Plan something, exit plan mode, run with `--dangerously-
skip-permissions`. That was the loop.

The thing to understand about me at this point: I'm a Solutions Architect,
not a software engineer. I hadn't really used GitHub on a day-to-day
basis. I wasn't tracking work in issues. I wasn't thinking in terms of
PRs and reviews. Each "feature" was just a chat with Claude that ended
when the chat ended.

This stage taught me what Claude Code could do. It also taught me what
breaks when context bloats — sessions that started clean by hour two
were producing weaker code than sessions that started fresh. That
observation kept nagging at me.

## Stage 1: Phased and non-phased modes

I started watching workflow videos. The Ralph-Wiggum concept — make sure
every session starts with fresh context — clicked immediately. That
matched what I was already seeing.

I built the **non-phased mode** first: a single-session orchestrator
(`/implement`) that runs implementation → critic → security → testing in
order, pausing at three human checkpoints. This was good for small to
medium changes — fast turnaround, clear gates, predictable.

Then I built the **phased mode** for bigger work: `/implement-phase`,
`/critic-phase`, `/security-phase`, `/test-phase`, `/commit-phase`. Each
runs in its own fresh Claude Code session. Close VS Code between phases.
Open it again. Run the next phase. Each phase gets a clean 200K-token
budget, no compactions, no late-session quality drop.

This was the bulk of my workflow on ChickenDuck (my main project) for
months. It worked. Issues moved from idea to merged code with discipline.
The structured handoff between phases — every state captured in
`.claude/implementations/issue-{N}.md` — meant I could pick up where I
left off after days away.

The cost: a lot of friction. Closing and re-opening sessions. Manually
running each phase command. Watching the issue file evolve. It was
robust but it required attention.

## Stage 2: The dual-model conversation

Somewhere on YouTube I heard the argument: "Claude tends to take the
easy route with tests it writes itself — it'll write tests that confirm
the implementation rather than the requirement." Half-truth-but-real,
exactly the sort of thing that's hard to falsify but easy to spot once
you start looking.

The pitch was: use a different model to score the work. Two models in
the loop. The implementation model and the review model can't be the
same one.

I started experimenting with Codex (GPT) as the second model. The
hybrid workflow that fell out of those experiments is what this kit
ships now:

- **Claude Code** does `/scope` (codebase exploration, alignment
  questions via `AskUserQuestion`), `/build` (implementation + tests),
  and `/verify` (test suite, manual test plan, commit).
- **Codex** does `/architect` (designs the implementation plan from
  Claude's scope) and `/review` (critic + security review of Claude's
  implementation).

Five phases. Two models. Switch IDEs at each handoff. The
`issue-{N}.md` file accumulates everything — Original Requirements,
Codebase Exploration, Architecture Plan - Attempt N, Implementation
Notes - Attempt N, GPT Review - Attempt N, Verification & Commit. The
file is append-only; no model overwrites another's work.

Every issue runs a build → review loop until Codex stamps **APPROVE**.
Then verify. Then commit.

Six months in, the only times this has produced something wrong are:

1. I wasn't specific enough at `/scope` time
2. I went too deep into scoping, then changed my mind partway through
   because the UX implications I'd already decided on turned out to
   demand a different implementation

The first is a discipline problem (scope harder up front). The second
is a real-world problem (sometimes the act of scoping reveals you
scoped the wrong thing). Neither is the workflow's fault. The workflow
just amplifies what I bring to it.

## Where I came in

I'm only really required at two places:

1. **`/scope`** — the alignment phase. Claude invokes `AskUserQuestion`
   extensively, and I make product / UX / scope-boundary calls. This is
   the single most leveraged time I spend. Half an hour here saves
   hours of rework downstream.

2. **`/verify`** — the gate phase. Claude generates a manual test plan,
   I run it in a browser, sign off, commit.

Everything between scope and verify — architect → build → review,
review → build → review until APPROVE — runs without me beyond
switching IDEs.

## Skills, not wholesale installs

Three other things that influenced this kit, none of them installed
wholesale:

- **BMAD** (Breakthrough Method for AI-Driven Development) — a 3-phase
  greenfield inception framework. I didn't install the whole BMAD
  toolchain; I asked Claude to absorb the _idea_ (Analyst →
  PM → Architect for new projects) and write `/bmad-start` and
  `/bmad-synthesize` commands that fit my existing
  `PROJECT_DOCUMENTATION_STANDARD.md` shape. Same outputs, integrated
  with my workflow.

- **superpowers** — a Claude Code plugin with curated skills (output
  styles, brainstorming patterns, debugging recipes, prompt patterns).
  I cherry-picked techniques into existing commands rather than
  installing wholesale. Most of its value is the patterns, not the
  installer.

- **RTK** (Rust Token Killer) — a CLI proxy that rewrites common dev
  commands transparently. 60–90% token savings on read-only operations
  (`git status`, `ls`, etc.). Set-and-forget. Not a workflow change,
  just a token bill change.

- **caveman** — an output style that strips filler and articles from
  chat without touching technical substance. ~75% reduction on
  conversational output. I have a "documentation = no caveman" rule
  in my global CLAUDE.md so workflow-output files (scope docs, plans,
  reviews) stay in normal prose; only chat is compressed.

## Where I'm headed

The current pain point: I'm still firing each command myself. I run
`/scope`, then I switch to Codex and type `/architect`, then I switch
back and type `/build`, then back to Codex for `/review`, then back to
Claude for `/verify`. The loop between `/build` and `/review` happens
several times per issue. Each time I'm just typing "check the latest
documentation in the issue file" and routing the result back.

The next step is orchestration: a thin layer that watches the
`issue-{N}.md` file, knows which model owns the next section, and
fires the appropriate command in the appropriate IDE without me. I
only get pulled in at `/scope` and `/verify` — the two places where
human judgment is actually load-bearing. I have a working name for
this layer ("Openclaw") but no implementation yet. It's the next
project, not this one.

## What this kit is and isn't

This kit captures the workflow as it exists in April 2026. It's
robust, daily-driven, and battle-tested on production projects. It is
**not** the orchestration layer above. That's still ahead.

If you're earlier in your Claude Code journey than I am: the
non-phased mode (`/implement`) is the right starting point. The
phased mode is for when you start hitting compactions on bigger work.
The hybrid mode is for when you're ready to commit to two models and
want the highest quality bar. Promote yourself through the stages as
your needs grow.

If you're at roughly the same stage as me, this kit should be a
near-drop-in for what I run today. The differences will be your
project's `AGENTS.md` (the project-specific overlays — your stack,
your constraints, your checks) and your `CLAUDE.md` (your conventions
and dev commands). Everything else is generic.

The kit's value isn't novelty. It's the structured shape of the
workflow itself, plus the discipline of keeping the universal
`/architect` and `/review` contract in one shared file rather than
duplicating it across every project. That single refactor — extracting
the shared standard from individual project AGENTS.md files —
shrunk per-project setup substantially and made cross-project
improvements possible. It's the most recent change in this kit and
probably the most important one for someone scaling beyond a single
project.
