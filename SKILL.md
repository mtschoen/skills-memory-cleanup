---
name: memory-cleanup
description: Use when the user asks to clean up, audit, prune, consolidate, or do maintenance on their agent memory / notes corpus - triggers include "memory cleanup", "clean up my notes", "audit the notes corpus", "prune stale memory", "consolidate memory notes", "run the memory linter". Drives the `replica memory lint` audit and works the findings category by category, with a mandatory fact-survival check before any deletion. Requires the `replica` CLI to be installed and pointed at the notes corpus.
---

# Memory corpus cleanup

The notes corpus is a flat directory of one-fact-per-file markdown notes with
YAML frontmatter (`name`, `description`, `type`), indexed by a generated
`MEMORY.md`. `replica memory lint` audits it and reports findings; this skill
works those findings category by category. The linter never mutates - you do,
deliberately, and you commit at the end.

**The single most important rule:** a note flagged "stale" because it names a
renamed/archived/migrated project is usually NOT dead - its underlying fact
often still applies to the migrated code under outdated paths. Verify
fact-survival against the current code before deleting. Default to TRIM (fix
the paths, keep the fact), not delete. This is the step a naive pass skips,
and it loses real knowledge.

## Setup

Confirm the CLI is installed before doing anything else:

```bash
command -v replica >/dev/null || { echo "replica not found - install the replica CLI before proceeding" >&2; exit 1; }
```

If `replica` is missing, STOP and tell the user to install the `replica` CLI
(the schoen-lab `replica` package, which ships as part of
[schoen-lab](https://github.com/mtschoen/schoen-lab)) and point it at the
notes corpus. Do not hand-replicate the linter's audit logic
(orphan/stale-line/desc-drift/etc. detection) as a substitute - those checks
are the reason this skill declares the dependency instead of bundling its
own.

Run the audit and read the categories:

```bash
replica memory lint        # exits non-zero when findings exist (expected)
```

Findings come in these kinds (from `memory_lint.py`):

| kind | meaning | default action |
| --- | --- | --- |
| `orphan` | note on disk but missing from MEMORY.md | reindex |
| `stale-line` | MEMORY.md lists a note that no longer exists | reindex |
| `desc-drift` | index line != frontmatter `description` | reconcile, then reindex (see below) |
| `malformed` | missing `type` and/or `description` frontmatter | add frontmatter |
| `taxonomy` | `type:` value not in the schema | reclassify |
| `stale-note` | RESOLVED/DONE/FIXED/SHIPPED/SUPERSEDED/COMPLETE marker in the head | review + prune (heuristic, many false positives) |
| `duplicate` | two notes share an identical `description` | merge or differentiate |
| `cluster` | >=5 notes share a topic token | triage (a "go look" prompt, NOT "merge all") |

Valid `type` values: `user`, `feedback`, `project`, `idiom` (or `idioms`),
`spike`, `gotcha`, `reference`. Filename prefixes route an untyped note to a
section: `reference_*`->reference, `spike_*`->spike, `idioms_*`->idiom,
`project_*`/`feature_*`/`handoff_*`->project, `feedback_*`->feedback,
`user_*`->user, `gotcha_*`->gotcha. The References section is the schema
fallback, so a note whose prefix the schema does not list (e.g. `plan_*`)
lands there unless it sets an explicit `type:`.

## Phase 1 - mechanical fixes (low risk, no content loss)

Do these first; they are unambiguous wins and shrink the noise for later phases.

1. **desc-drift.** Do NOT blindly `reindex` - the index line is regenerated
   from frontmatter, but humans/agents often hand-enrich the index line so it
   is *richer* than the frontmatter. For each drifted note, compare the two:
   if the index line carries more/fresher detail, copy it into the frontmatter
   `description` first. Then `replica memory reindex` aligns every index line
   to frontmatter in one shot. (Spot-check a few drifts before deciding - if
   the index is uniformly equivalent, a straight reindex is fine.)
2. **malformed.** Read each note; add `type` (from the list above) and a
   one-line, information-dense `description` (this string drives recall
   relevance - make it specific, not generic). The linter accepts both a
   top-level `type:` and the same key nested under a `metadata:` block
   (top-level wins); match each note's existing style rather than rewriting it.
   This is high-volume and mechanical: fan out to subagents (one batch of
   ~7 notes each) rather than doing it inline.
3. **taxonomy.** Change the offending `type:` to the correct schema value, or
   (if the value is genuinely a new category the corpus wants) extend
   `memory_schema.toml` in the replica package instead.
4. `replica memory reindex`, then `replica memory lint` again - desc-drift,
   malformed, taxonomy should all be 0.

## Phase 2 - fact-survival check (run per-candidate before any prune in Phase 3/4)

Before deleting any prune candidate, verify whether its fact still holds
against the current code/docs. The common hard case is a note flagged stale
for naming a renamed/archived/migrated project - for those, build the
migration map first (what moved where). For each candidate inspect the
relevant current code/docs and assign:

- **DEAD** - pure ephemeral state, OR the fact is contradicted/superseded by
  current code. Safe to delete. (Confirm against the actual code, not the
  note's own claims - e.g. grep for the file/feature it says is unbuilt.)
- **LIVE (KEEP/TRIM)** - the gotcha/requirement/worklist still applies. KEEP if
  paths are fine; TRIM to fix outdated paths/names and drop the dead sections,
  preserving the surviving fact.

Default to LIVE when unsure. This check routinely flips a majority of
"obviously stale" candidates from delete to trim. It also catches triage
false-positives (e.g. a flagged "outdated claim" that is actually correct once
you read both notes) - verify before you "fix."

Fan out the verification (it is read-heavy); the trims themselves are good
subagent work given a precise keep/trim spec per note. Phase 3 and Phase 4
below both surface prune candidates - run this check against each one *before*
any `git rm`, not after.

## Phase 3 - prune stale-notes (judgment + deletion)

`stale-note` is a keyword heuristic with a high false-positive rate. Read each
candidate's head and classify:

- **Genuinely dead** -> prune (git-recoverable). e.g. one-shot session intent,
  a dated handoff whose work shipped/merged, a recap full of in-flight job IDs,
  a self-expired note ("delete this when X").
- **False positive -> keep.** Durable feedback that merely contains "DONE" in
  prose; an active project note with open work; a note explicitly retained as
  a reusable recipe; a note that now documents *current* behavior despite a
  SUPERSEDED header.

Run the Phase 2 fact-survival check on every "genuinely dead" candidate first.
Then present the prune list to the user (grouped keep/prune with one-line
reasons) and get approval before deleting. Deletes go through `git rm`.

## Phase 4 - consolidate clusters (heaviest judgment)

A `cluster` finding means N notes share a topic token. It is a prompt to LOOK,
not a mandate to merge. The corpus is **one-fact-per-file by design**; most
clusters are legitimately distinct facts that should stay separate. Coarse
tokens (e.g. a shared filename prefix) produce false clusters of unrelated
notes - never merge those.

Triage read-only first. Fan out subagents (one per cluster, or a few clusters
each), each instructed to find ONLY genuine redundancy and report:

- **MERGE GROUPS** - true duplicates or tightly-overlapping fragments of the
  *same* fact. Merge by folding all unique content into one note (keep the
  better-named file), then `git rm` the other. Preserve every unique detail.
- **STALE/PRUNE** - dated handoffs/recaps surfaced during the read. Run these
  through the Phase 2 fact-survival check before deleting.
- **KEEP SEPARATE** - the (usual) majority; one line is enough.

Expect few real merges. The real redundancy in a mature corpus is stale
session-state notes, not duplicate facts.

## Finish

1. `replica memory reindex`
2. `replica memory lint` - confirm malformed/taxonomy/desc-drift are 0. Some
   `cluster` and `stale-note` findings will remain: legitimate topical
   groupings kept separate by design, and deliberate keeps with a
   false-positive keyword. That is the healthy steady state, not a failure.
3. Sanity scan the diff: no HTML-entity leakage (`&lt;`/`&gt;`) in frontmatter
   from subagents; no note left without `type`/`description`.
4. Commit (one commit per phase reads well: mechanical+prune, then
   consolidation). The corpus is git-managed; `replica memory push` (or the
   SessionEnd hook) syncs it.

## Orchestration notes

- **Main agent owns judgment**, subagents own volume. Keep merges, the prune
  recommendation, and the fact-survival verdicts with the orchestrator; fan
  out malformed-frontmatter batches, cluster triage, verification, and trims.
- **Permissions:** subagent edits to the corpus directory can silently deny if
  it sits outside the workspace allowlist. Probe with one batch and confirm the
  edits actually landed (`git status`) before scaling the fan-out.
- **No em-dashes** in note content; write literal `<`/`>` (not HTML entities).
- Per-edit aislop/quality hooks may run on note writes - a score-100 result is
  a useful signal the frontmatter is well-formed.

## What this skill is NOT

- Not a mass-merge tool. "One fact per file" is the design; resist collapsing
  topical clusters.
- Not an excuse to delete on a keyword match. The fact-survival check is the
  whole point.
- Not for project repos - that is the `project-maintenance` skill. This is
  specifically the agent notes/memory corpus.
