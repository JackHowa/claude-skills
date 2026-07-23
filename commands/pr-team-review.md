# PR Team Review

Run a multi-lens panel review of a single PR: several agents each review the same diff through a different, specific lens, run in parallel, then get synthesized into one combined write-up. Use this deliberately for a PR that warrants more than one kind of eye on it — it is not the default review loop.

## When to use this instead of `/review-mine`

`/review-mine` is the fast, single-pass loop for working through your assigned-PR queue. Reach for this skill instead when a specific PR would benefit from dedicated attention beyond a general code review — for example: it touches a design system or component library and deserves a design-system-compliance pass, or it closes out a tracked ticket and you want the ticket's documentation drafted from the actual shipped diff at the same time as the review. Don't run this on routine PRs; it's heavier and opt-in.

## Arguments

`$ARGUMENTS` identifies the PR: a full PR URL, or `{number} {owner}/{repo}`. If ambiguous or missing, ask which PR before doing anything else.

## Preflight

- Requires `gh` CLI. If missing, stop and tell the user.
- `GH_TOKEN` must be valid. Run `gh auth status` — if it fails, stop and ask the user to rotate the token.
- If a ticket-documentation lens will run (see below) and it needs an external tracker (e.g. Jira via an Atlassian MCP connection), check that connection is authenticated first. If it isn't, still run the other lenses and just skip the ticket lens with a clear note rather than failing the whole review.

## Fetch once, shared across lenses

Fetch the diff, PR metadata, and CI status a single time and pass them into every agent below — don't have each lens agent re-fetch the same data:

```bash
gh pr diff {number} --repo {owner}/{repo}
gh pr view {number} --repo {owner}/{repo} --json title,body,author,additions,deletions,changedFiles,commits
gh pr checks {number} --repo {owner}/{repo}
```

## Run the panel

This skill's invocation is itself the explicit opt-in to use the `Workflow` tool for multi-agent orchestration — go ahead and use it here without asking again.

Determine which lenses apply to this PR and run all applicable ones as parallel `agent()` calls in a single `parallel()` (or a `pipeline()` if a lens depends on another's output — none do by default):

- **Correctness & security lens** (always runs): the same checklist as `/review-mine` — logic bugs, edge cases, security issues, the crash-risk performance checks (uncaught errors, unmemoized effect/memo dependencies, unstable default params, leaked listeners/subscriptions), clarity, nits.
- **Design-system-compliance lens** (runs only if the diff touches UI component, style, or design-token code): checks for hardcoded values that should reference design tokens, deviation from established component/variant patterns, and accessibility issues (missing labels, keyboard operability, focus handling, contrast-dependent logic). Skip entirely for non-UI PRs — don't force this lens onto a backend or infra diff.
- **Ticket-documentation lens** (runs only if the PR body references a tracked ticket AND the tracker connection is available): drafts a concise, ticket-ready summary of what actually shipped, grounded in the diff itself rather than restating the PR description — call out anything the diff does that the ticket doesn't yet mention, and anything the ticket describes that the diff doesn't actually do. This lens **drafts only** — never post or update the ticket without an explicit go-ahead (see Present step below).

Each lens should return its findings as structured output (schema: findings list with severity, or for the ticket lens, the drafted text) so they can be merged cleanly rather than three separate walls of prose.

## Synthesize and present

Merge the lenses into one review, in the same format `/review-mine` uses:

```
### [{title}]({url})
**Repo:** {owner}/{repo} | **Author:** {author} | **Changes:** +{additions} -{deletions}

**Summary:** ...

**Findings:**
a. 🔴 [BLOCKER] ...
b. 🟠 [SUGGESTION] ...
c. 🔵 [NIT] ...
d. 🟢 [NOTE] ...

**Verdict:** `APPROVE` / `REQUEST CHANGES` / `COMMENT`
```

Letter findings (a, b, c...), never number or bullet them. Tag every finding with its color-emoji severity — 🔴 `[BLOCKER]`, 🟠 `[SUGGESTION]`, 🔵 `[NIT]`, 🟢 `[NOTE]` — scoped to the findings list only, no emoji elsewhere. HTML-escape any raw angle brackets in `{title}`. If two lenses raised essentially the same point, merge them into one finding rather than listing it twice.

If the ticket-documentation lens ran, show its draft in a separate section after the verdict:

```
**Ticket update draft:**
...
```

Then use `AskUserQuestion` for the review action (same as `/review-mine`: Approve / Approve with comment / Skip / Stop, with `rc`/`c` as typed fallbacks), and — only if a ticket draft was produced — a **separate** follow-up question asking whether to post/update the ticket with that draft, post it verbatim only, edited, or not at all. Never post to the tracker without that explicit confirmation, even if the PR review itself is approved.
