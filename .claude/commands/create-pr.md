---
description: Create a pull request for the current branch using this repo's configured defaults
---

Create a pull request for the current branch. Extra context from the user, if any: $ARGUMENTS

**Step 0 — Load repo defaults** (do this before anything else, including preflight): look for `.claude/pr-defaults.json`.

- If it exists, read it for this repo's defaults (`merge_flow`, `default_base_branch`, `hotfix_base_branch`, `default_reviewers`, `jira_base_url`, `jira_cloud_id`, `jira_project_key`, `jira_code_review_transition`). This repo's merge flow is one-way: feature branch → each branch in `merge_flow`, in order (e.g. `dev` → `stage` → `main`). Use `default_base_branch` as the PR base unless the user says this is a hotfix (use `hotfix_base_branch` instead) or explicitly names another base branch.
- If it's missing, tell the user it doesn't exist yet, that creating it is optional but lets future runs skip the base-branch/reviewer/Jira questions, and show them exactly where to put it and what it can contain:

  > No `.claude/pr-defaults.json` found in this repo. It's optional, but without it I'll ask for the base branch every time and skip default reviewers and Jira linking. To set it up, create `.claude/pr-defaults.json` with any of these keys (all optional — omit what you don't use):
  >
  > ```json
  > {
  >   "merge_flow": ["dev", "stage", "main"],
  >   "default_base_branch": "dev",
  >   "hotfix_base_branch": "main",
  >   "default_reviewers": ["github-username1", "github-username2"],
  >   "jira_base_url": "https://your-org.atlassian.net",
  >   "jira_cloud_id": "your-cloud-id",
  >   "jira_project_key": "ABC",
  >   "jira_code_review_transition": "Code Review"
  > }
  > ```
  >
  > - `merge_flow` — ordered branch chain feature branches flow through.
  > - `default_base_branch` — branch PRs target by default (usually the first entry in `merge_flow`).
  > - `hotfix_base_branch` — branch to target instead for hotfixes (usually `main`).
  > - `default_reviewers` — GitHub usernames auto-requested as reviewers on every PR.
  > - `jira_base_url` — Atlassian site URL, used to build ticket links.
  > - `jira_cloud_id` — Atlassian cloud ID, needed for Jira transition calls via MCP.
  > - `jira_project_key` — Jira project prefix (e.g. `ABC` for `ABC-1234`), used to detect ticket keys.
  > - `jira_code_review_transition` — Jira status to move a ticket to once its PR is created (e.g. `"Code Review"`).

  Then proceed without it for this run: ask the user for the base branch instead of guessing, and skip default reviewers and Jira linking — don't invent any of it. If the user wants the file created now, offer to do it, but don't block the rest of this command on that — continue creating the PR once you have the base branch.

**Preflight** (check before anything else besides Step 0): run `gh auth status` and `git remote -v`.
- If `gh auth status` fails, tell the user to run `gh auth login` and stop — nothing downstream will work without it.
- If there's no `origin` remote, tell the user to add one (`git remote add origin <url>`) and stop.

**Protected-branch guard**: run `git branch --show-current`.
1. If the current branch is NOT one of the branches listed in `merge_flow`, skip this section entirely and proceed normally — you're already on a feature branch.
2. If it IS one of `merge_flow` (the user is sitting directly on `dev`/`stage`/`main`), a PR cannot be opened from that branch into itself. Check what's actually pending: uncommitted/staged changes (`git status`), plus any local commits not on the remote (`git log origin/<branch>..HEAD`, if it has an upstream).
   - If there's nothing pending (clean tree, no local-only commits), tell the user there's nothing to open a PR for and stop — don't create an empty branch.
   - Otherwise, ask the user for the Jira ticket key (see Jira ticket resolution below — resolve it here, don't ask twice) and confirm they want a new branch created off the current state with the pending changes committed there. Propose a branch name such as `<ticket-key-lowercase>-<kebab-summary-of-the-change>` and let them override it before creating anything.
   - On confirmation: `git switch -c <new-branch>` — this carries both the uncommitted changes and any local-only commits onto the new branch without touching the old branch's history — then stage and commit the pending changes using this repo's `type: description` commit convention. If it's not obvious what should be staged or how to phrase the message, ask rather than guessing.
   - If the protected branch had local-only commits before this, ask whether to reset the local `<branch>` pointer back to match `origin/<branch>` now that those commits are safely preserved on the new branch. This rewrites local branch state, so it needs explicit confirmation — never do it automatically, and skip asking if there was nothing to reset.
   - From this point on, treat the new branch as the current branch for the rest of this command.

**Jira ticket resolution** (needed so this PR shows up on the ticket's Development panel):
1. Look for a `<jira_project_key>-<number>` pattern (case-insensitive) in the branch name, then in `$ARGUMENTS`, then in the commits being included.
2. If still not found, ask the user for the ticket key before creating the PR — don't invent one, and don't silently skip it (the whole point is every PR ends up linked to its ticket).
3. Once resolved, normalize to uppercase (e.g. `halo-1234` → `HALO-1234`).
4. The GitHub-for-Jira integration links a PR to a ticket by matching the issue key in the branch name, commit messages, or **PR title** — a link in the body alone will not make it show up on the ticket. So the key must end up in the title, not just the description.

Steps:

1. Run in parallel:
   - `git status` (untracked files)
   - `git diff` (unstaged changes) and `git diff --cached` (staged changes)
   - Check whether the current branch tracks a remote and is up to date with it
   - `git log <base>..HEAD` and `git diff <base>...HEAD` (where `<base>` is the resolved base branch) to see every commit and change that will be included

2. If `git log <base>..HEAD` comes back empty (no commits ahead of `<base>` at all), this branch has nothing to open a PR for — warn the user the PR would be empty and confirm before continuing, rather than opening a no-op PR.

4. Analyze ALL commits that will be included (not just the latest one) and draft:
   - A PR title matching this repo's actual commit convention: lowercase `type: short description`, with the resolved Jira key appended in parentheses, under 70 characters total, e.g. `fix: cache invalidation on av creation (HALO-1234)`. `type` is one of `feat`, `fix`, `chore`, `refactor`, `docs`, `perf`, `test`, or `update` (the repo's de facto catch-all for changes that aren't a clean feat/fix) — pick whichever the commits in this branch mostly are. Do not use the legacy all-caps prefix (`FEAT:`, `FIX:`) — it's not what this repo actually does anymore.
   - A body with:
     - A `Jira: [<KEY>](<jira_base_url>/browse/<KEY>)` line at the very top, before the summary
     - `## Summary` — 1-3 bullet points on what changed and why

5. Push the branch to origin with `-u` if it isn't already tracked/up to date (ask first if this would force-push or overwrite remote history — e.g. after an amend or rebase on a branch that already has an open PR). This alone is enough to update an existing PR's diff; GitHub reflects new/amended commits automatically once pushed, no further action needed for that part.

6. Check whether an open PR already exists for this branch: `gh pr view --json number,url,title,body 2>/dev/null`.
   - If one exists, this is an update, not a new PR — skip `gh pr create` entirely (it would just error with "a pull request for this branch already exists").
   - Regenerate the title/description to reflect the accumulated changes and update it with `gh pr edit <number> --title "<type>: <description> (<KEY>)" --body "$(cat <<'EOF' ... EOF)"`, same format as a new PR (Jira link line, `## Summary`). The PR description is owned by its author, not reviewers, so refreshing it on every run is the expected default here — no need to ask first.
   - Skip straight to step 8 (Jira transition) and then return the existing PR's URL — don't fall through to step 7.
   - If no open PR exists for this branch, continue to step 7 as normal.

7. Create the PR against the resolved base branch with `gh pr create`, passing the body via a HEREDOC and `--reviewer` for each entry in `default_reviewers` (omit `--reviewer` entirely if the list is empty or the file is missing — never invent reviewers):

   ```
   gh pr create --base <base> --title "<type>: <description> (<KEY>)" --reviewer <user1> --reviewer <user2> --body "$(cat <<'EOF'
   Jira: [<KEY>](<jira_base_url>/browse/<KEY>)

   ## Summary
   - ...
   EOF
   )"
   ```

   If `gh pr create` errors on a reviewer (e.g. reviewer is the PR author, or lacks repo access), retry once without `--reviewer` and tell the user which reviewer(s) to add manually.

8. If `jira_code_review_transition` and `jira_cloud_id` are set and the Atlassian MCP tools are available, move the resolved ticket to that status now that the PR exists:
   - Call `getTransitionsForJiraIssue` with `cloudId: <jira_cloud_id>` and `issueIdOrKey: <KEY>`.
   - Find the entry whose `name` matches `jira_code_review_transition` case-insensitively, and use its `id`.
   - Call `transitionJiraIssue` with that `cloudId`, `issueIdOrKey`, and `transition: { id: <matched id> }`.
   - Don't hardcode a transition ID — always resolve it fresh from the lookup, since IDs aren't guaranteed stable across issue types.
   - If the named transition isn't in the list (ticket already past that column, workflow differs for this issue type, tools not connected), don't fail the whole command — tell the user the PR was created but the ticket wasn't moved, and why. This is also the expected outcome on an update to an already-existing PR whose ticket has already made this transition — not an error, just nothing to do.

9. Return the PR URL.

Do not use `--no-verify` or skip hooks. Do not force-push. Never merge directly into any branch listed in `merge_flow` without a PR, and never target `hotfix_base_branch` directly unless the user says this is a hotfix.
