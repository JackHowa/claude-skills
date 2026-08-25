---
name: pr-status
description: Reports on the user's own open GitHub pull requests — what's mergeable right now, what's blocked and why (failing CI, merge conflicts, behind base, changes requested, pending review, draft), grouped by Jira ticket with a short explanation of what each ticket is about. Use whenever the user asks "how are my PRs doing", "what can I merge", "what's the status of my PRs", "anything of mine ready to merge", or similar check-in questions about their own open pull requests — even if they don't name a specific repo or ticket. Also use for merge requests like "merge the approved ones" or "merge <X>" once this skill has already established which PRs are ready.
allowed-tools: Bash, mcp__claude_ai_Atlassian__getJiraIssue
---

# PR Status by Ticket

Find the user's own open PRs, work out where each one actually stands, and group the
results by Jira ticket so the status is self-explanatory rather than a bare list of
numbers. The point isn't just "here are your PRs" — it's "here's what needs your
attention and why."

## Preflight

- Requires `gh` CLI. If missing, stop and tell the user.
- Get the authenticated user: `gh api user --jq .login`

## Find open PRs

```bash
gh search prs --author=<login> --state=open --json repository,title,url,number,isDraft --limit 100
```

Exclude personal repos — anything under the user's own GitHub account rather than a work
org. These clutter a work status check and the user doesn't want them mixed in even if
they technically match the author filter.

## Check status for each PR

For each remaining PR:

```bash
gh pr view <number> --repo <owner>/<repo> --json mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,isDraft
```

The `statusCheckRollup` summary alone isn't enough to tell a real failure from an
irrelevant pending job — pull the actual check names and conclusions:

```bash
gh pr checks <number> --repo <owner>/<repo>
```

Classify each PR:
- **Mergeable** — `reviewDecision` is `APPROVED`, not a draft, and no real CI job is
  failing. A pending/on-hold visual-regression job (e.g. Percy on hold) does not by
  itself block merge — mention it, but don't treat it as a blocker on its own.
- **Blocked — CI failing** — approved, but a real test/build job shows `fail`/`FAILURE`.
- **Blocked — branch behind** — approved and checks pass, but `mergeStateStatus` is
  `BEHIND` (needs updating from base before it can merge).
- **Blocked — conflicts** — `mergeable` is `CONFLICTING` or `mergeStateStatus` is `DIRTY`.
- **Pending review** — `reviewDecision` is `REVIEW_REQUIRED`.
- **Changes requested** — `reviewDecision` is `CHANGES_REQUESTED`.
- **Draft** — `isDraft` is true.

If a PR that showed up in an earlier check this conversation is no longer in the open
list, don't just drop it silently — the user cares whether it merged or got closed:

```bash
gh pr view <number> --repo <owner>/<repo> --json state,mergedAt
```

## Group by ticket

Parse the leading ticket key from the PR title (e.g. `WBC-3048: ...`). PRs without one
(e.g. `adhoc: ...`, `chore: ...`) go in a "No ticket" group, further split by repo if
there's more than one. A PR that references another PR/ticket for context in its title
(e.g. "verify EDS beta ... (EDS #1020)") still gets its own key if the title has one;
otherwise fold it under the ticket of the change it's verifying, if that's stated.

## Explain each ticket

For every distinct ticket key, fetch its summary so the grouping means something on its
own rather than just being a label:

```
mcp__claude_ai_Atlassian__getJiraIssue with cloudId "expel-io.atlassian.net" and issueIdOrKey "<TICKET-KEY>"
```

Use `fields.summary` — one line, not the full description, unless the user asks for more
detail on a specific ticket.

If the Atlassian tool is unavailable or unauthenticated, don't let that block the rest of
the report. Say once, briefly, that ticket summaries couldn't be refreshed and the user
needs to authenticate the Atlassian connector (via claude.ai connector settings, or
`/mcp` in an interactive session), then fall back to whatever ticket context is already
known from the conversation, or just the bare ticket key.

## Output

```
## <TICKET-KEY> — <one-line summary, omit if unavailable> ([ticket](https://expel-io.atlassian.net/browse/<TICKET-KEY>))
- <status emoji> **<status>**: [<repo>#<number>](<url>) — <short reason>
```

If anything changed since a prior check earlier in the same conversation (newly merged,
newly closed, newly conflicted, a test that got fixed or newly broke), call that out —
that delta is often the most useful part of the update.

Close with a one-line bottom line: exactly which PRs are mergeable right now, or "nothing
mergeable" if none are.

Always link both the ticket and the PR — never bare numbers or unlinked text.

## Merging

If the user then asks to merge specific PRs (e.g. "merge the approved ones", "merge the
auth-ui one"), re-verify each one's current state right before merging — status can go
stale between the report and the action:

```bash
gh pr view <number> --repo <owner>/<repo> --json mergeable,mergeStateStatus,reviewDecision,isDraft
```

Only merge PRs that are still classified **Mergeable**. Use `gh pr merge <number> --repo
<owner>/<repo> --squash` unless the user specifies a different merge strategy, then
confirm the merge landed:

```bash
gh pr view <number> --repo <owner>/<repo> --json state,mergedAt
```
