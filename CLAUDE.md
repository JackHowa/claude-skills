## Communication Style
- Answer exactly what was asked; do not volunteer Slack drafts, extra summaries, or commentary unless requested.
- Show the actual changed lines — a diff or the code with `file:line` references — rather than only describing the change, so I can validate it without opening the file.
- When making performance or comparison claims, always show the raw numbers and the arithmetic — never round to marketing-style figures like '200-300x faster'.
- Never assume a person's pronouns; use their name or 'they' unless I have told you otherwise.

## Git & PR Conventions
- NEVER base branches or PRs off `main`. Always branch from and target `develop` unless I explicitly say otherwise.
- Exception: the design system repo and infrastructure repos use `main` as their base branch — branch from and target `main` there. If unsure which convention a repo uses, check its default branch and recent merge history before branching.
- After changing an implementation mid-PR, always update the PR description and title to match the final approach before asking for review.
- Before opening a PR, run typecheck, lint, and the test suite locally and report the results in the PR body. Re-run before merging, and watch for test regressions you introduced yourself.
- Add reviewers only once CI checks pass, not at PR open time. Always add the workbench-core team as a reviewer.
- Once a fix is verified, open the PR directly — don't ask for confirmation first.

## Scope Discipline
- Change only the files needed for the stated task. If a fix appears to require touching infra, nginx, prod config, or unrelated apps, STOP and ask before editing.
- Prefer the narrowest possible diff; split unrelated changes into separate PRs rather than bundling them.
- Clean up only your own mess: remove imports/variables/functions that YOUR changes made unused, but leave pre-existing dead code alone — mention it instead of deleting it.
- Match existing style, even if you'd do it differently. Don't "improve" adjacent code, comments, or formatting.
- Before committing, re-read the full diff and justify every changed line against the task's stated scope; revert anything incidental (lockfile noise, unrelated deletions). Never pop a stash you didn't create.

## Simplicity First
- Search for an existing implementation before writing your own — grep the codebase for the helper, component, hook, or util, and check shared/design-system packages. Prefer reusing or extending what's there over hand-rolling a duplicate; if you do write new code, say what you looked for and why it didn't fit.
- Minimum code that solves the problem. No speculative features, no abstractions for single-use code, no unrequested configurability, no error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

## Goal-Driven Execution
- Turn tasks into verifiable goals: "fix the bug" → "write a test that reproduces it, then make it pass"; "refactor X" → "tests pass before and after".
- For multi-step work, state a brief plan as `step → verify: check` before starting, then loop until each check passes.
- Before editing, list assumptions you're under 90% confident on — which flag gates this, which user segment, where a control lives, current vs. hypothetical state — and confirm them before writing code.
- State the deliverable type before acting — ticket-only, plan-only, or code — and don't exceed it without asking, even if more seems obviously needed.
- When a plan spans more than one Jira ticket or touches CI/build config, the plan must explicitly state how the work splits into PRs/tickets and which CI layer (job vs. step vs. script) it belongs in — don't leave deliverable shape implicit and default to bundling or the first layer that works.

## Diagnosis Before Fixes
- State the root cause and the evidence for it — failing job/file/line, a minimal repro, expected vs observed — before writing any fix.
- If you can't reproduce it, say so and give your top two hypotheses with the test that would distinguish them.
- Never report a check (lint/test/pre-commit) as passed unless you saw its actual output — don't filter out or ignore a tool crash/warning in the process.

## Jira Conventions
- Jira comments/descriptions must use proper ADF/markup — never emit literal `h2.` wiki markup or literal `\n` escape sequences; verify the formatting renders.
- Ticket descriptions lead with an **Acceptance Criteria** section written as clear bullet points, followed by a **Dev Notes** section for engineering-focused detail (implementation approach, affected files/services, gotchas).
- Keep updates concise and do only what was asked unless extra output is requested.
- When closing work: link the PR in the ticket, then transition the ticket in one step without asking for extra confirmation.

## First-Principles Algorithm
"The most common mistake of smart engineers is to optimize a thing that should not exist." Run these steps in order; don't skip ahead:
1. **Question the requirements.** Requirements are always dumb to some degree, however smart the person who gave them. Otherwise you get the perfect answer to the wrong question.
2. **Try to delete the part or process step entirely.** If you're not forced to put back at least 10% of what you delete, you're not deleting enough. Needing to restore nothing means you were too conservative.
3. **Optimize or simplify** — only after trying to delete.
4. **Speed it up** — only after deleting and optimizing; otherwise you're speeding up something that shouldn't exist.
5. **Automate** — last. Automating first means you may automate, speed up and simplify something you later delete.
