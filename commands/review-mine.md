# Review Code Assigned to Me

Find open PRs where I'm requested as a reviewer, fetch the diffs, and produce a thorough code review for each.

## Preflight

- Requires `gh` CLI. If missing, stop and tell the user.
- `GH_TOKEN` must be valid. Run `gh auth status` — if it fails, stop and ask the user to rotate the token.

## Find PRs

```bash
gh search prs --review-requested=@me --state open --json number,title,url,repository,updatedAt,author --limit 50
```

Filter out PRs from archived repos:

```bash
gh api repos/{owner}/{repo} --jq '.archived'
```

Skip any where `archived` is `true`.

## Arguments

If `$ARGUMENTS` is provided, treat it as a filter: only review PRs whose title or repo name contains the argument (case-insensitive). If empty, review all open PRs found.

## Release PRs

Jack's external auto-approve routine for release-branch PRs (titles like `Release/`, `release/`, `v\d`, `deploy/`) is currently down — do NOT skip these and do NOT silently auto-approve them. Review them like any other PR, but lighter: they're typically an aggregate cut bundling several already-individually-reviewed feature PRs from `develop` into `main` (or similar), so a fresh line-by-line re-review of the bundled diff usually isn't warranted. Instead:

- Note in the summary that it's a release cut and list the bundled tickets/PRs (from the PR body).
- Check CI status (`gh pr checks {number} --repo {owner}/{repo}`) and surface any failing checks as a finding — don't wave them through unexamined.
- Still go through the normal `AskUserQuestion` action flow below — do not auto-approve without asking.

If Jack confirms the external routine is back, ask whether to restore the old skip/auto-approve behavior.

## Generate reviews in parallel

Once the PR list is finalized (archived repos filtered out), draft every review concurrently instead of one at a time: spawn one `Explore` agent per remaining PR, all in a single message so they run in parallel. Give each agent:

- the PR's `number`, `repo`, and `title`
- instructions to fetch the diff (`gh pr diff {number} --repo {owner}/{repo}`), PR metadata (`gh pr view {number} --repo {owner}/{repo} --json title,body,author,additions,deletions,changedFiles,commits`), and CI status (`gh pr checks {number} --repo {owner}/{repo}`)
- the release-PR lighter-treatment instructions above, if titled like a release cut
- the full review checklist below (Correctness/Security/Performance/Clarity/Nits) and the exact `Output format` further down, so the agent's final text *is* the finished, ready-to-post review for that PR

Wait for all agents to return before presenting anything. This only parallelizes the drafting — decisions still happen one at a time (see "Present each PR" below); nothing gets posted to GitHub until you weigh in on that specific PR.

Each agent should produce a review covering:
   - **Summary** — what the PR does in 2-3 sentences
   - **Correctness** — logic bugs, edge cases, off-by-ones, unhandled errors
   - **Security** — injection, auth issues, exposed secrets, unsafe inputs
   - **Performance** — N+1 queries, unnecessary work, missing indexes. Flag as [BLOCKER] if any of the following crash-risk issues are present (high bar — do not block on style or minor correctness):
     - Likely uncaught TypeErrors or null dereferences that would throw and crash a component
     - Infinite re-render loops — new object/array/function created inline as a `useEffect` or `useMemo` dependency with no memoization, causing it to re-run every render
     - Default parameter values that are object/array/function expressions (e.g. `prop = []`, `prop = {}`, `prop = new Date()`) passed into a hook dependency array — creates a new reference every render
     - Memory leaks from event listeners or subscriptions added without a cleanup function
   - **Clarity** — confusing names, missing context, dead code
   - **Nits** — minor style issues (clearly labeled, non-blocking)

## Present each PR — one at a time

Once every agent has returned, walk through the PRs in the original order. For each one, output the agent's finished review verbatim (re-checking it briefly against the diff if it looks thin, contradicts the PR description, or the verdict seems off — don't blindly relay a weak agent output), then use the `AskUserQuestion` tool to present a single-select choice before moving to the next PR. This decision step is sequential regardless of how the reviews were generated — only one PR is ever being approved/commented/skipped at a time.

## Output format

Output the review for one PR, then use the `AskUserQuestion` tool to present a single-select choice (no typing required):

```
question: "What would you like to do with {title}?"
header: "Action"
options:
  - label: "Approve"         description: "Approve the PR silently"
  - label: "Approve with comment" description: "Approve and add a comment"
  - label: "Skip"            description: "Move to next PR without action"
  - label: "Stop"            description: "End session and summarize"
```

Note: `AskUserQuestion` supports max 4 options. Request changes and comment are rare — if needed, the user can type "rc" or "c" as free text and you handle it. Accept typed shortcuts as a fallback at any time.

If the user selects or types `Approve` / `a`, run:
```bash
gh pr review {number} --repo {owner}/{repo} --approve
```

If the user selects or types `Approve with comment` / `ac`, ask for the comment body, then approve with `--body`.

If the user types `rc` / `request changes`, ask for the comment body, then request changes.

If the user types `c` / `comment`, ask for the comment body, then post a comment.

If the user selects or types `Skip` / `s`, move to the next PR without taking any action.

If the user selects or types `Stop` / `q`, end the session and summarize what was reviewed.

Format each review as below. Do NOT print any `---` horizontal rules (they render as stray `---{title}` frontmatter-like lines in the terminal). Separate reviews with the heading alone — no leading or trailing rule.

### [{title}]({url})
**Repo:** {owner}/{repo} | **Author:** {author} | **Changes:** +{additions} -{deletions} | **PR {n} of {total}**

Before using the `[{title}]({url})` heading, HTML-escape any raw angle brackets in `{title}` (e.g. a PR titled `...after_clone_<table>...` must render as `after_clone_&lt;table&gt;`). An unescaped `<word>` inside markdown link text breaks out of the `[...]()` syntax in some renderers and can swallow the rest of the message into what looks like one giant link. This applies to every title in the session, not just the first one.

**Summary:** ...

**Findings:**
a. 🔴 [BLOCKER] ...
b. 🟠 [SUGGESTION] ...
c. 🔵 [NIT] ...
d. 🟢 [NOTE] ...

**Verdict:** `APPROVE` / `REQUEST CHANGES` / `COMMENT`

Letter findings (a, b, c...), never number or bullet them — numbers collide with PR-selection numbering when the user references a finding while multiple PRs are in play. When the user references a letter later (e.g. "note b in the comment", "approve with comments from a and c"), map it back to that finding's text.

Every lettered item MUST carry a category tag with its color emoji prefix — 🔴 `[BLOCKER]`, 🟠 `[SUGGESTION]`, 🔵 `[NIT]`, or 🟢 `[NOTE]` (for a positive/confirming observation that isn't an issue at all). Never present an untagged or emoji-less finding, even a positive one — this applies consistently across every PR in the session, not just the first one. Do NOT use the emoji anywhere else in the review (not in the summary, not in the verdict) — keep it scoped to the findings list only, so it stays a scannable severity signal rather than decoration.

For large/complex PRs, keep findings to a short list of the most important issues rather than an exhaustive enumeration — favor a clear, digestible breakdown over completeness.

If no PRs are found, say so clearly.

## Learning the user's review style

This is a standing goal across sessions, not a one-time step: build up a picture of how the user actually reviews PRs — what they approve outright, what they insist on commenting/blocking on, and what they let slide as non-blocking.

Whenever the user takes an action that reveals a preference (approves despite a flagged finding, insists on a specific comment, requests changes, skips a PR, or corrects how you presented something), save a short memory capturing what mattered to them and why (if stated) — following the memory system's `feedback` type conventions (rule, **Why:**, **How to apply:**). Don't wait until the end of the session to do this — capture it in the moment, right after the decision.

### Learned review bar (defaults observed from past sessions)

These are durable defaults distilled from the user's actual review decisions — apply them, but keep refining as new sessions reveal exceptions:

- **Aggregate/release-style PRs** (a cut that bundles several already-individually-reviewed commits into another branch): light touch. Verify CI is green and skim the bundled list; don't re-review the underlying diffs line-by-line. Approve unless CI is failing.
- **Dependency-bump-only PRs** (no consumer code changes, just a version bump): approve readily. A failing visual-regression/screenshot-diff check on this kind of PR is worth a one-line note (it's routine for a version bump to shift a few pixels) but isn't by itself a reason to withhold approval — flag it, don't block on it.
- **Real logic/architecture problems in substantive code are a hard blocker even when CI is green.** CI passing doesn't validate a design choice CI can't check (e.g. two implementations of the same concept behaving subtly differently). Conversely, when CI is legitimately red on substantive (non-doc, non-bump) code, don't approve blind — hold at a non-approving verdict until it's resolved or you understand why.
- **Defer to a teammate's already-posted substantive review when it's more thorough than your own pass.** Read it, verify its claims against the diff rather than rubber-stamping either the PR or the teammate's take, and if it holds up, write agreement concisely rather than re-deriving the same analysis from scratch.
- **Stray/unrelated changes** (leftover debug code, an accidental unrelated file edit, a hardcoded test value) are comment-worthy nits, not blockers on their own — approve with a comment calling them out rather than blocking merge over them.
- **Comment tone: terse and friendly.** No filler, no hedging, no restating the obvious back to the author.
- **Never guess at an unresolved CI failure's root cause without evidence.** If you can't reach the actual logs/output, say so plainly instead of presenting a guess as a diagnosis.

## Follow-up: stale reviews awaiting response

After all PRs have been reviewed (or when the user stops the session), run a follow-up check for open PRs where you left feedback that hasn't been addressed.

```bash
gh search prs --reviewed-by=@me --state open --json number,title,url,repository,author --limit 50
```

For each result (skipping archived repos and any PRs already reviewed this session), fetch the review and comment history:

```bash
gh pr view {number} --repo {owner}/{repo} --json reviews,comments,commits \
  --jq '{reviewDecision, myLastReview: [.reviews[] | select(.author.login=="{gh_username}")] | last, latestCommit: (.commits | last), comments: .comments}'
```

Surface a PR only if **all** of the following are true:
- Your last review is `CHANGES_REQUESTED` or `COMMENTED` (not `APPROVED`)
- The author has not pushed a new commit **or** replied with a comment since your review
- The PR has been idle for more than 7 days

For each stale PR, show a one-line summary:

> **[{title}]({url})** *(author, {days} days idle)* — your last action: {state} — "{review body snippet}"

Then ask via `AskUserQuestion`:
```
question: "What would you like to do with {title}?"
options: ["Ping author", "Close PR", "Skip", "Stop"]
```

If `Ping author`: post a comment — `@{author} Checking in — is this still in progress, or should it be closed?`

If `Close PR`: close with — `Closing due to no activity since {date}. Feel free to reopen if you pick this back up.`

If `Skip` or `Stop`: move on.
