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

## Auto-approve releases

If a PR title matches common release branch patterns (e.g. `Release/`, `release/`, `v\d`, `deploy/`) — check its CI status:

```bash
gh pr checks {number} --repo {owner}/{repo}
```

If all checks pass (or there are no checks), approve it automatically without pausing:

```bash
gh pr review {number} --repo {owner}/{repo} --approve
```

Print a one-line confirmation: `✓ Auto-approved release PR #{number} ({title})` and move on.

If any checks are failing, treat it as a normal PR and pause for input.

## Review each PR — one at a time

Work through PRs one at a time. For each PR:

1. Fetch the diff:
   ```bash
   gh pr diff {number} --repo {owner}/{repo}
   ```

2. Fetch PR description and metadata:
   ```bash
   gh pr view {number} --repo {owner}/{repo} --json title,body,author,additions,deletions,changedFiles,commits
   ```

3. Read the diff carefully and produce a review covering:
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

**Summary:** ...

**Findings:**
- [BLOCKER] ...
- [SUGGESTION] ...
- [NIT] ...

**Verdict:** `APPROVE` / `REQUEST CHANGES` / `COMMENT`

If no PRs are found, say so clearly.

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
