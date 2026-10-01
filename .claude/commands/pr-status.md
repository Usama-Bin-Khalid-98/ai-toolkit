---
description: Show your open PRs and the PRs waiting on your review across all your GitHub repos, grouped by what needs doing next (failing CI, changes requested, conflicts, ready to merge, stale)
---

Report the state of open pull requests and what each one needs next. Scope or extra context from the user, if any: $ARGUMENTS

This command isn't tied to any one project. It searches every repo your GitHub account can see, so it works best installed once in `~/.claude/commands/` and run from anywhere.

## Data required

Personal defaults, kept inline so this command doesn't need a separate config file. Edit the values below directly, or let the command prompt you and fill them in itself the first time it runs.

- `stale_after_days`: *(not set, optional — days without activity before a PR is flagged as stale; 3 if unset)*
- `jira_base_url`: *(not set, optional — Atlassian site URL, used to link each PR's ticket, e.g. https://your-org.atlassian.net)*
- `jira_cloud_id`: *(not set, optional — Atlassian cloud ID, needed only to show each ticket's current Jira status)*

**Step 0 — Resolve the data above** (do this before anything else, including preflight): every value here is optional, so ask for all the unset ones in a single question, and don't block on any of them. Once the user answers, edit this file (`pr-status.md`) and replace each bullet with the value they gave. For anything they skip, replace the bullet with `*(skipped)*` so future runs don't ask again. Without the Jira fields, leave out ticket links and statuses (see the reduced mode at the end).

**Preflight**: run `gh auth status`, then `gh api user --jq .login` to get the user's GitHub login. Use this login in place of `@me` in the search queries below.
- If `gh auth status` fails, tell the user to run `gh auth login` and stop.

## 1. Work out the scope

Build one or two GitHub search queries from `$ARGUMENTS`. Every query starts with `is:pr is:open archived:false`.

**Whose PRs** (default: both of the first two):
- your PRs: `author:<login>`
- PRs requesting your review: `review-requested:<login>`. This includes requests made to a team you're on.
- `mine`: only your PRs. `reviews`: only PRs requesting your review.
- `@<username>`: that person's PRs (`author:<username>`).

**Where** (default: everywhere your account can see):
- `here`: only the current repo. Get it with `gh repo view --json nameWithOwner` and add `repo:<owner/name>`. If the current directory isn't a GitHub repo, say so and stop.
- `org:<name>` or `repo:<owner/name>`: pass it straight into the query.

These can be combined, e.g. `reviews org:acme` or `mine here`.

**A single PR URL** (or a number together with `here`): report just that PR, with the full detail from step 3 and nothing grouped. Fetch it with `gh pr view <url> --json number,title,url,author,headRefName,baseRefName,isDraft,createdAt,updatedAt,reviewDecision,reviewRequests,latestReviews,mergeable,mergeStateStatus,statusCheckRollup`, plus the review-thread count from the GraphQL query below.

## 2. Fetch the PRs

Run one GraphQL search per query, in parallel. Each call returns everything the report needs, so you don't need a separate call per PR:

```
gh api graphql -f q='<search query>' -f query='
query($q:String!){
  search(query:$q, type:ISSUE, first:50){
    issueCount
    nodes{ ... on PullRequest{
      number title url isDraft createdAt updatedAt
      author{ login }
      repository{ nameWithOwner }
      headRefName baseRefName
      reviewDecision mergeable mergeStateStatus
      reviewRequests(first:20){ nodes{ requestedReviewer{ ... on User{ login } ... on Team{ slug } } } }
      latestReviews(first:20){ nodes{ author{ login } state submittedAt } }
      reviewThreads(first:100){ nodes{ isResolved isOutdated } }
      commits(last:1){ nodes{ commit{ committedDate statusCheckRollup{ state contexts(first:50){ nodes{
        ... on CheckRun{ name status conclusion }
        ... on StatusContext{ context state }
      } } } } } }
    } }
  }
}'
```

Use `-f` for the search string, not `-F`, so it's always sent as a string.

If `issueCount` is above 50, say how many PRs weren't shown and suggest narrowing the scope with `org:`, `repo:` or `here`.

If a PR appears in both searches (for example, a PR of yours where someone also requested you), report it once, as your PR.

Search results often come back with `mergeable: UNKNOWN`, because GitHub only computes it when a PR is viewed directly. For each non-draft PR with `UNKNOWN`, re-fetch it once with `gh pr view <url> --json mergeable,mergeStateStatus`, all in parallel. Viewing it is also what makes GitHub start the computation. If it's still unknown, report it as "merge state unknown" and don't guess.

## 3. Work out each PR's state

**CI**, from the last commit's `statusCheckRollup`:
- **Failing**: `state` is `FAILURE` or `ERROR`. Name the failing checks: check runs with a `conclusion` of `FAILURE`, `TIMED_OUT`, `CANCELLED` or `ACTION_REQUIRED`, and status contexts with a `state` of `FAILURE` or `ERROR`.
- **Running**: `state` is `PENDING` or `EXPECTED`.
- **Passing**: `state` is `SUCCESS`.
- **None**: `statusCheckRollup` is null. Report "no checks", not "passing".

**Review**, from `reviewDecision`, `latestReviews` and `reviewRequests`:
- `CHANGES_REQUESTED`: name who requested changes.
- `APPROVED`: name who approved.
- `REVIEW_REQUIRED`, or empty with reviewers still listed in `reviewRequests`: waiting on those reviewers.
- Empty with no reviewers requested: "no reviewers". This is worth flagging, because nobody is going to look at the PR.
- Unresolved threads: count the threads with `isResolved: false` and `isOutdated: false`, and include the count when it's above zero.

**Mergeability**, from `mergeable` and `mergeStateStatus`:
- `CONFLICTING`: merge conflicts with the base branch.
- `BEHIND`: the branch is behind the base and needs updating before it can merge.
- `BLOCKED`: blocked by branch protection (usually missing approvals or required checks). Say which reason applies from the CI and review state above.
- `CLEAN`: can be merged.

**Age**: days since `updatedAt`. A PR is **stale** when that's more than `stale_after_days` (3 if unset).

**Ticket**: look for a Jira-style key, i.e. 2–10 uppercase letters, a hyphen and a number (`ABC-123`, case-insensitive), in the title, then in the branch name. Normalize it to uppercase. Ignore matches that are clearly something else, such as `UTF-8`, `SHA-256` or `ISO-8601`.

## 4. Look up Jira statuses (optional)

Skip this step if `jira_cloud_id` is unset, no keys were found, or the Atlassian tools aren't connected.

Make one call for all the keys together, not one per PR:

```
searchJiraIssuesUsingJql
  cloudId: <jira_cloud_id>
  jql:     key in (<KEY-1>, <KEY-2>, ...)
  fields:  ["status","assignee"]
```

Jira rejects the whole JQL query if any key doesn't exist on that site, which can happen with keys from other organizations' repos. If the call fails for that reason, drop the keys the error names and retry once. Show the dropped keys as plain text with no link.

Flag mismatches between the PR and the ticket, for example a ticket still "In Progress" while its PR is approved, or a ticket already "Done" while its PR is still open. Report them. Never transition or edit the tickets.

## 5. Report

Group the PRs by what has to happen next. A PR goes in the first group that applies, so each PR appears only once:

1. **Needs your action** (your PRs): CI failing, changes requested, conflicts, behind base, or unresolved threads. Give the specific reason or reasons.
2. **Ready to merge** (your PRs): approved, CI passing, `CLEAN`.
3. **Waiting on others** (your PRs): review pending or CI running. Name who or what you're waiting on and for how long.
4. **Needs your review** (other people's PRs): sorted oldest request first. Mark PRs where you already reviewed and were re-requested ("re-review"), and PRs where changes you requested are still open.
5. **Drafts**: listed on one line each, with no further analysis.

Mark stale PRs with ⚠ in whichever group they're in, and show "no reviewers" as a reason in the group the PR lands in.

Show each group as a table with these columns: Repo (short name, or leave the column out when everything is in one repo), PR (number and title, linked), Ticket (linked when `jira_base_url` is set, plus its Jira status if known), Base, CI, Review, Age, and Next step. Keep "Next step" to a few words, e.g. "fix `lint` check", "resolve 3 threads", "rebase on dev", "merge", "nudge @alice". Leave out empty groups. If nothing is open in scope, say so in one line.

End with a one-line summary, e.g. "5 open across 3 repos: 2 need you, 1 ready to merge, 2 waiting. 3 reviews requested from you." Then offer at most two concrete follow-ups that fit what you found, such as updating a branch that's behind its base. Do any of them only if the user says yes.

**Reduced mode** (no Jira settings, or the Atlassian tools aren't connected): skip step 4, show ticket keys without links or statuses (or leave out the Ticket column if no keys were found), and say the Jira status wasn't checked.

## Pitfalls

- **Read-only.** This command only reads GitHub and Jira. It never merges, rebases, pushes, comments, re-requests reviewers, closes PRs or moves tickets unless the user agrees to a follow-up in step 5.
- **Search lag.** GitHub's search index can be a minute or two behind, so a PR opened or closed just now may be missing or still listed. For one exact PR, `gh pr view <url>` is always current.
- **Passing vs no checks.** A null `statusCheckRollup` means no CI ran. Don't report that as passing.
- **`mergeable: UNKNOWN` is temporary.** GitHub computes it lazily. Re-fetch once (step 2) before reporting it, and never report an unknown state as conflict-free.
- **Outdated threads.** Threads on lines that have since changed (`isOutdated: true`) usually don't need action. Don't count them as unresolved.
- **Approved but blocked.** An `APPROVED` decision doesn't mean the PR can merge. Required checks, a branch that's behind, or a second required approval can still block it. Base "Ready to merge" on `mergeStateStatus: CLEAN`, not on the approval alone.
- **Org access.** PRs in organizations that use SAML SSO only appear if the `gh` token is authorized for that org. If the user expects PRs that aren't showing up, suggest `gh auth refresh` and authorizing the token for that org.
