# Review Code Assigned to Me

Find open PRs where I'm requested as a reviewer, fetch the diffs, and produce a thorough code review for each.

## Preflight

- Requires `gh` CLI. If missing, stop and tell the user.
- `GH_TOKEN` must be valid. Run `gh auth status` — if it fails, stop and ask the user to rotate the token.

## Find PRs

```bash
gh search prs --review-requested=JackHowa --state open --json number,title,url,repository,updatedAt,author --limit 50
```

Filter out PRs from archived repos:

```bash
gh api repos/{owner}/{repo} --jq '.archived'
```

Skip any where `archived` is `true`.

## Arguments

If `$ARGUMENTS` is provided, treat it as a filter: only review PRs whose title or repo name contains the argument (case-insensitive). If empty, review all open PRs found.

## Auto-approve releases

If a PR title starts with `Release/` (case-insensitive), check its CI status:

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

Output the review for one PR, then stop and ask:
> **What would you like to do?** approve (`a`) / approve with comment (`ac`) / request changes (`rc`) / comment (`c`) / skip (`s`) / stop (`q`)

Wait for the user's response before moving to the next PR. Accept both the shortcut and the full word.

| Input | Action |
|---|---|
| `a` / `approve` | Approve silently |
| `ac` / `approve with comment` | Ask for comment body, then approve with it |
| `rc` / `request changes` | Ask for comment body, then request changes |
| `c` / `comment` | Ask for comment body, then post a comment |
| `s` / `skip` | Move to next PR, no action |
| `q` / `stop` | Summarize and end |

If the user says `a`, run:
```bash
gh pr review {number} --repo {owner}/{repo} --approve
```

If the user says `ac`, `rc`, or `c`, ask for the comment body, then run the appropriate `gh pr review` command with `--body`.

If the user says `s`, move to the next PR without taking any action.

If the user says `q`, end the session and summarize what was reviewed.

Format each review as:

---
### [{title}]({url})
**Repo:** {owner}/{repo} | **Author:** {author} | **Changes:** +{additions} -{deletions} | **PR {n} of {total}**

**Summary:** ...

**Findings:**
- [BLOCKER] ...
- [SUGGESTION] ...
- [NIT] ...

**Verdict:** `APPROVE` / `REQUEST CHANGES` / `COMMENT`

---

If no PRs are found, say so clearly.
