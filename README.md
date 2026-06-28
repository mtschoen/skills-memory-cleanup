# skills-memory-cleanup

A Claude Code / Agent Skill that drives a careful cleanup pass over the agent
**memory / notes corpus** - the flat directory of one-fact-per-file markdown
notes indexed by a generated `MEMORY.md`.

The skill is built on the `replica memory` command surface (the `replica`
package in schoen-lab): `replica memory lint` audits the corpus and reports
findings by kind (orphan, stale-line, desc-drift, malformed, taxonomy,
stale-note, duplicate, cluster); the skill works those findings category by
category and commits the result. The linter never mutates - the agent does,
deliberately.

## Why it exists

The central, easy-to-skip guardrail is the **fact-survival check**: a note
flagged "stale" for naming a renamed/archived/migrated project usually still
carries a live fact under outdated paths, so the correct action is to TRIM (fix
the paths, keep the fact), not delete. A naive keyword pass loses real
knowledge. The skill encodes that check plus the corpus's one-fact-per-file
principle, which keeps `cluster` findings from triggering destructive
mass-merges.

It is distinct from the sibling skills: `project-maintenance` (project repos,
not the notes corpus), `wrap` (saves memory at session end), and
`reconcile-tasks` (PLAN.md vs git). None covers the corpus-linter cleanup loop.

## Layout

- `SKILL.md` - the skill (the only file the installer ships).

`README.md` is dev-only and is not installed (the skills-dev installer ships a
top-level allowlist: `SKILL.md` plus `scripts/`, `references/`, `assets/`, and
anything declared in a `.skillpack`).

## Requirements

The `replica` CLI must be installed and pointed at the notes corpus. The skill
deliberately stays dependency-light and replica-driven - no bundled scripts in
v1.
