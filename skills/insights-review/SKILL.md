---
name: insights-review
description: Runs /insights, compares its friction categories against the last saved baseline, and turns recurring or new friction into concrete CLAUDE.md convention additions the user approves one at a time. Use whenever the user asks to "run insights", "check for recurring friction", "see if I'm improving", "update my CLAUDE.md based on insights", or wants to compare a fresh insights report against a previous one. Also trigger proactively right after generating an /insights report if the user asks how it compares to before, or whether anything should go in CLAUDE.md.
allowed-tools: Bash, Read, Edit, Write, Skill
---

# Insights Review

The `/insights` report tells you what's actually going wrong in a repeatable way —
friction categories with counts, specific incidents, and quotes of corrections the user
had to repeat. Most of that signal decays the moment the report is read. This skill
closes the loop: it diffs today's friction against the last time this ran, and turns
anything that's recurring or newly costly into a specific, approvable CLAUDE.md edit —
the same way a manual review session would, just without needing the user to remember
to ask.

The output of one run becomes the input to the next: every run ends by saving a new
baseline memory, so the next run has something concrete to diff against.

## Step 1 — Get the current report

If an `/insights` report already ran earlier in this conversation, reuse that data
instead of re-running it. Otherwise invoke the `/insights` skill/command to generate a
fresh one.

Pull out `friction_analysis.categories` (name, description, examples) and, if present,
any numeric friction counts mentioned in `interaction_style` or `at_a_glance` (e.g.
"wrong_approach (42)", "buggy_code flagged in 21 sessions"). These numbers are the
comparable signal across runs — the prose descriptions change wording every time even
when the underlying problem hasn't moved.

## Step 2 — Find the last baseline

Baselines live in the user's memory system as files named like
`project_insights_baseline_<date>.md` (see
`/Users/jackhoward/.claude/projects/-Users-jackhoward-sites-claude-skills/memory/`).
Check `MEMORY.md` in that directory for the most recent one — there should only ever be
one live baseline; older ones get superseded, not accumulated.

If no baseline exists yet, skip the diff — this run itself becomes the first baseline
(and Step 3 has nothing to compare against, so go straight to proposing conventions for
whatever friction categories are one-off but structural, i.e. the kind that would recur
next month).

## Step 3 — Diff against the baseline

Build a small table: category → last count/status → this run's count/status → trend
(new / recurring / resolved / worse). A category counts as "recurring" if it shows up
again even under different wording — e.g. "wrong Jira project key" and "wrong Jira label
casing" are the same underlying gap (unverified Jira conventions) even if the report's
prose differs run to run. Use judgment here, not string matching.

Show this table to the user before proposing anything. It's the part that answers "am I
actually improving" — that's usually why this skill got invoked in the first place.

## Step 4 — Propose CLAUDE.md additions, one at a time

For each category that is recurring, worse, or newly significant (appeared with
multiple concrete examples, not a single fluke), check whether the existing
`~/.claude/CLAUDE.md` already covers it:

1. Read `~/.claude/CLAUDE.md` (and any project-level `CLAUDE.md` in play, if the friction
   is project-specific rather than global).
2. If a section already addresses the gap, say so explicitly and move on — don't
   propose a duplicate. A near-miss (e.g. the file has a general "run tests" rule but
   the actual incident was about a masked exit code hiding failures) is not a duplicate;
   flag the specific gap.
3. If the same incident recurred *after* a rule was already added for it — the rule
   exists, covers the case, and still didn't prevent the repeat — adding more prose is a
   no-op; the rule already had its chance. Say so explicitly and propose escalating
   instead: a hook that checks automatically (e.g. a pre-commit/pre-PR script), or a
   per-person/per-case memory note if the failure is about retaining a specific
   correction rather than following a general rule. Don't just restate the rule louder.
4. If it's a genuine gap, draft the smallest addition that would have prevented the
   specific incident — not a broad policy essay. Cite the incident briefly so the "why"
   is traceable later. Prefer adding to an existing section over creating a new one
   unless nothing fits.

Present proposals numbered, each as:
```
N. [section] "<exact line to add>"
   why: <one-line incident this would have prevented>
```

Ask the user to approve/skip per item — don't bulk-apply. Apply each approved line with
Edit immediately after it's approved, the same way manual review sessions in this
project have worked, rather than batching edits to the end.

## Step 5 — Save the new baseline

After the user has gone through all proposals (whether they approved 0 or all of them),
write a fresh memory file superseding the old baseline:

- Name it `project_insights_baseline_<today's date>.md`.
- Record the current friction counts/categories (the comparable numbers from Step 1),
  which CLAUDE.md additions were applied this round and why, and which proposed
  additions the user explicitly declined (so a future run doesn't re-propose something
  already rejected without new evidence).
- Update `MEMORY.md`'s index line to point at the new file, and remove the old
  baseline's index line (delete the superseded memory file itself, don't leave stale
  duplicates).

This is what makes the next run's diff in Step 3 meaningful — without it, every run
would look like "first run" again.

## Notes

- This skill is about steering CLAUDE.md, not about relitigating whether the report's
  framing is fair. If the user disagrees with how `/insights` characterized something,
  that's feedback for the report itself, not a reason to force a CLAUDE.md edit.
- Keep proposed lines terse and imperative, matching the existing CLAUDE.md voice (short
  bullets under `##` sections, not paragraphs).
- If the friction is clearly project-specific (e.g. only ever happens in one repo), the
  addition belongs in that project's own `CLAUDE.md`, not the global
  `~/.claude/CLAUDE.md`. Ask if it's unclear which scope fits.
