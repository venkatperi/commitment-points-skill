---
name: commitment-points
description: >
  Run a structured constraint discovery pass BEFORE writing code that introduces
  (1) a new service, Lambda, API endpoint, or LLM call; (2) a new piece of
  user-facing state such as preferences, consent, votes, session data; or (3) a
  new business event type such as signups, purchases, submissions, credit
  movements. Surfaces platform limits, state persistence decisions, and analytics
  capture points that become expensive rewrites when missed. Trigger on planning
  phrases like "let's add", "I'll implement", "next we'll build", "let's wire
  up", "let's build", or any request that would introduce one of the three
  change types. Do NOT trigger for pure refactors, styling changes, copy edits,
  tests, or changes that don't affect service topology, state location, or
  event emission.
---

# Commitment Points

Three triggers fire this skill. For each trigger, load the matching reference and produce a commitment check artifact BEFORE generating implementation code.

## Triggers

| If the change introduces | Class | Load |
|---|---|---|
| A new service, Lambda, API endpoint, LLM call, or external hop | Boundary | `references/boundary-checkpoint.md` |
| A new piece of user-facing state | Continuity | `references/continuity-checkpoint.md` |
| A new business event type | Capture | `references/capture-checkpoint.md` |

Multiple triggers can fire on one feature. Load all that apply.

## Flow

1. Identify which triggers fire. State them at the top of your response: `Triggers fired: boundary, capture.`
2. Load the relevant reference file(s) with the `view` tool.
3. Produce the commitment check artifact using `templates/artifact.md`.
4. Write the artifact to file (file mode) or render it inline (inline mode). See mode selection below.
5. Wait for user acknowledgment or revision before generating implementation code.

Do not collapse steps. Do not generate implementation code in the same turn as the artifact unless the user explicitly instructs "proceed without review" or equivalent.

## Mode selection: file vs inline

At the start of each skill run, check whether `.commitment-points/` directory exists in the project root.

- **Directory exists → file mode.** Write the artifact to `.commitment-points/checks/<YYYY-MM-DD>-<feature-slug>.md`. Create `checks/` if missing. In chat, print a one-line summary plus the file path.
- **Directory does not exist → inline mode.** Print the full artifact in the chat response.

The user opts into file mode by running `mkdir -p .commitment-points/checks` in the project root. Zero-config for exploration; auditable for serious projects.

Do not attempt to create `.commitment-points/` yourself. That is the user's opt-in.

## Project policy suggestion

On the first skill run of a session, check whether the project has a `CLAUDE.md` at the root that references commitment-points. If it does not, suggest (once per session, not every invocation) that the user add this line:

> Before writing any code that introduces a new service, a new piece of user state, or a new event type, run the commitment-points skill and produce the constraint artifact. Do not skip to implementation without this document.

Frame it as an optional hardening step. Skills loaded via description are best-effort pattern matching. CLAUDE.md directives are binding. The user should know the difference.

## When NOT to run

Skip the skill for:

- Pure refactors that preserve service topology, state location, and event emission
- Styling, layout, copy changes
- Bug fixes to existing code paths
- Test additions
- Tooling, build config, deployment config without new services or endpoints
- Renaming, moving files, import reorganization

When in doubt, ask the user one sentence: *"This looks like a <category> change — should I run the commitment check?"* Do not over-apply. A skill that fires on every turn is a skill the user will disable.

## Output discipline

The artifact is not a retrospective. It is written before code and used to decide the shape of the code. If any field in the artifact is `?`, `unknown`, or `TBD`, the check is not complete. Either:

- Name the measurement the user needs to take, or
- Name the doc the user needs to read, or
- Mark the item as `assumption: X — revisit if Y`

Do not generate implementation code while unresolved unknowns remain, unless the user explicitly acknowledges them.

## Output brevity

The artifact uses tables by design. Do not pad with prose. A good artifact is scannable in 30 seconds. If a section needs paragraphs of explanation, the issue belongs in a follow-up discussion, not the artifact.

## Worked examples

See `examples/askprasna-failures.md` for three real commitment-point misses, each with the artifact that would have caught them and the rewrite that followed from skipping the check. Reference these when the current feature mirrors one of the patterns.
