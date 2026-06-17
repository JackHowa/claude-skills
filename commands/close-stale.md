# Close Stale Approved PRs

Find open PRs where I've already approved and previously commented asking the author to take action. If a nudge comment exists and the PR is still inactive, close it.

## Preflight

- Requires `gh` CLI. If missing, stop and tell the user.
- `GH_TOKEN` must be valid. Run `gh auth status` — if it fails, stop and ask the user to rotate the token.

## Find approved PRs

Search for open PRs where the authenticated user has left an approval review:

```bash
gh search prs --reviewed-by=@me --state open --json number,title,url,repository,updatedAt,author --limit 100
```

Filter out archived repos:

```bash
gh api repos/{owner}/{repo} --jq '.archived'
```

Skip any where `archived` is `true`.

## Arguments

If `$ARGUMENTS` is provided, treat it as a filter: only consider PRs whose title or repo name contains the argument (case-insensitive). If empty, check all approved open PRs.

## For each PR, check staleness and prior nudge

1. Fetch comments to check if a nudge was already posted:
   ```bash
   gh api repos/{owner}/{repo}/issues/{number}/comments --jq '[.[] | {author: .user.login, body: .body, created_at: .created_at}]'
   ```

2. A "nudge comment" exists if there is a comment from `@me` that:
   - Asks if the PR is still active / in progress
   - Mentions closing or inactivity
   - Was posted more than 7 days ago

3. Check when the PR was last updated (`updatedAt`). A PR is stale if it has had no activity for **30+ days**.

4. Classify each PR into one of:
   - **Close candidate** — nudge comment exists AND no activity since the nudge AND PR is 30+ days stale
   - **Nudge candidate** — PR is 30+ days stale but no nudge comment yet
   - **Active** — updated within 30 days, skip silently

## Output — one PR at a time

For each **Close candidate**, show:

---
### [{title}]({url})
**Repo:** {owner}/{repo} | **Author:** {author} | **Last activity:** {updatedAt}
**Your nudge:** "{first 100 chars of nudge comment}" ({nudge date})

> **Close this PR for inactivity?** yes (`y`) / skip (`s`) / stop (`q`)

---

If the user says `y`, close with a comment:
```bash
gh pr comment {number} --repo {owner}/{repo} --body "Closing due to inactivity — no response since my earlier nudge. Feel free to reopen if this work picks up again."
gh pr close {number} --repo {owner}/{repo}
```

If the user says `s`, move to the next PR.

If the user says `q`, end the session and summarize.

## After close candidates — nudge candidates

Once all close candidates are handled, show any **Nudge candidates** (stale but no prior nudge):

---
### [{title}]({url})
**Repo:** {owner}/{repo} | **Author:** {author} | **Last activity:** {updatedAt}

> **Post a nudge?** yes (`y`) / skip (`s`) / stop (`q`)

---

If the user says `y`, post:
```bash
gh pr comment {number} --repo {owner}/{repo} --body "Hey — this PR has been inactive for 30+ days. Is it still in progress, or should it be closed? (Nudge from @me)"
```

## Summary

At the end, output:
- **Closed:** list each PR title + URL
- **Nudged:** list each PR title + URL
- **Skipped:** count
