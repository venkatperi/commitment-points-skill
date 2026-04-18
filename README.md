# commitment-points

A Claude Code skill that runs a structured constraint discovery pass before you introduce new services, new user state, or new business events.

Catches the three classes of decisions that cost a rewrite when missed: boundary (platform limits), continuity (where state lives), and capture (what gets recorded at event time).

## Install

Drop the folder into either location:

- **User-global:** `~/.claude/skills/commitment-points/` — available in every project
- **Project-local:** `<project-root>/.claude/skills/commitment-points/` — available only in this project

```bash
# User-global
mkdir -p ~/.claude/skills
cp -r commitment-points ~/.claude/skills/

# Or project-local
mkdir -p <project>/.claude/skills
cp -r commitment-points <project>/.claude/skills/
```

Verify Claude Code sees it: start a session in a project and ask *"What skills do you have available?"* The skill should appear in the list with its description.

## Usage

### Zero-config (inline mode)

Just plan work normally. When you say *"let's add a feedback rating feature,"* Claude loads the skill, identifies which triggers fire, produces a commitment check artifact, and renders it in chat. You approve or revise before any code is written.

### File mode (recommended for serious projects)

Opt in by creating the checks directory in your project root:

```bash
mkdir -p .commitment-points/checks
```

Once this directory exists, every artifact gets written to `.commitment-points/checks/<YYYY-MM-DD>-<feature-slug>.md`. The file is version-controllable and reviewable in pull requests. In chat, Claude prints a one-line summary plus the file path.

### Explicit invocation

If the skill doesn't fire on its own (description didn't match), force it:

```
/commitment-points before we build this
```

Or just: *"Run the commitment check before writing code."*

### CLAUDE.md directive (strongest enforcement)

Skills loaded via description are best-effort. Skills enforced via `CLAUDE.md` are binding. Add this to your project's `CLAUDE.md`:

```markdown
## Commitment checks

Before writing any code that introduces a new service, a new piece of user state, or a new event type, run the commitment-points skill and produce the constraint artifact. Do not skip to implementation without this document.
```

Claude Code reads `CLAUDE.md` at session start and treats directives there as project policy.

## The three triggers

| Change introduces | Class | Question |
|---|---|---|
| A new service, Lambda, API endpoint, LLM call, external hop | **Boundary** | Which hop dies first at worst-case latency? |
| A new piece of user-facing state | **Continuity** | Same user, new device tomorrow — what should they see? |
| A new business event type | **Capture** | What will I need to know about this in a month? |

## What the skill does NOT trigger on

- Pure refactors that preserve topology, state location, and event emission
- Styling, layout, copy changes
- Bug fixes to existing code paths
- Test additions
- Tooling, build config, CI changes without new services

## Structure

```
commitment-points/
├── README.md                      — this file
├── SKILL.md                       — entry point, routing logic, mode selection
├── references/
│   ├── boundary-checkpoint.md     — platform limits, request-path walkthrough
│   ├── continuity-checkpoint.md   — state location matrix, device-switch rule
│   └── capture-checkpoint.md      — atomic event writes, month-from-now test
├── templates/
│   └── artifact.md                — the check output format
└── examples/
    └── askprasna-failures.md      — three worked failures with artifacts
```

## Why this skill exists

Most build sessions mix two kinds of decisions. Most are reversible in an afternoon. A small subset embed in places that cost a full rewrite to unwind: service topology, state persistence, event capture. AI coding assistants compress the timeline between decision and consequence, so these commitment points are hit faster than they used to be.

The skill is a 90-second pause at three specific moments. It is not a replacement for judgment. It is a discipline that forces the right question at the right time.

## Related reading

See `examples/askprasna-failures.md` for three concrete failures, each costing hours of rework, that this skill would have caught in minutes.
