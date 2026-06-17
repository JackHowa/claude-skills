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

## Review each PR

For each PR:

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
   - **Performance** — N+1 queries, unnecessary work, missing indexes
   - **Clarity** — confusing names, missing context, dead code
   - **Nits** — minor style issues (clearly labeled, non-blocking)

## Output format

For each PR, output:

---
### [{title}]({url})
**Repo:** {owner}/{repo} | **Author:** {author} | **Changes:** +{additions} -{deletions}

**Summary:** ...

**Findings:**
- [BLOCKER] ...
- [SUGGESTION] ...
- [NIT] ...

**Verdict:** `APPROVE` / `REQUEST CHANGES` / `COMMENT`

---

If no PRs are found, say so clearly.
