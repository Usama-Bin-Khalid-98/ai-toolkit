---
description: Write QA notes for a PR, branch or set of commits, publish them as a doc, attach it to the Jira ticket and hand it to QA
---

Write QA notes for a change and get them to QA. What to cover, from the user, if any: $ARGUMENTS

## Data required

This repo's QA-notes defaults, kept inline so this command doesn't need a separate config file. Edit the values below directly, or let the command prompt you and fill them in itself the first time it runs.

- `jira_base_url`: *(not set — Atlassian site URL, used to build ticket links, e.g. https://your-org.atlassian.net)*
- `jira_cloud_id`: *(not set — Atlassian cloud ID, needed for every Jira call via MCP)*
- `jira_project_key`: *(not set — Jira project prefix, e.g. ABC for ABC-1234, used to detect ticket keys)*
- `default_reference_branch`: *(not set — branch a feature branch is diffed against when no reference branch is given, e.g. dev)*
- `qa_contact`: *(not set — the QA person who gets the notes, as "Name <email>"; mentioned on the Jira ticket and named in the share reminder)*
- `qa_ready_transition`: *(not set, optional — Jira status to move the ticket to once the notes are attached, e.g. "Ready for QA")*

**Step 0 — Resolve the data above** (do this before anything else): for any value still marked *(not set — ...)*, ask the user for it now — skip the ones marked optional if they have nothing to give. Once they answer, edit this file (`qa-notes.md`) and replace each bullet above with the value they gave, so future runs don't ask again. If the user doesn't use Jira for this repo, leave the three Jira fields unset and run in chat-only mode (see step 6) instead of blocking. If `default_reference_branch` is unset, ask for the reference branch whenever a branch is given without one.

## 1. Work out what the notes are about

The user names the change in `$ARGUMENTS` in one of these forms. Use the first one that matches:

- **PR**: a number (`#123`, `123`) or a GitHub PR URL. Run `gh pr view <n> --json number,title,body,url,headRefName,baseRefName,commits` and `gh pr diff <n>`. If `gh auth status` fails, tell the user to run `gh auth login` and stop.
- **Branch, with or without a reference branch**: e.g. `feature/abc-12-login` or `feature/abc-12-login against stage`. Run `git fetch origin` first, then `git log <ref>..<branch> --no-merges` and `git diff <ref>...<branch>`, where `<ref>` is the reference branch the user named or else `default_reference_branch`. Use `origin/<name>` if a branch only exists on the remote.
- **Commits**: one or more SHAs, or a range (`a1b2c3..d4e5f6`). Run `git show --stat --patch <sha>` for each SHA, or `git log` + `git diff` over the range.
- **Nothing given**: use the current branch (`git branch --show-current`) against `default_reference_branch`. If the current branch *is* the reference branch, or the range is empty, ask the user what to cover. Don't guess.

If the range has no commits or no diff, say so and stop. Empty notes are no use to QA.

For a large diff, read `--stat` first, then read the files that change behaviour (handlers, UI, queries, migrations, config, feature flags). Lockfiles, generated code and formatting-only changes can be skipped.

## 2. Resolve the Jira ticket

Skip this step in chat-only mode.

1. Look for a `<jira_project_key>-<number>` pattern (case-insensitive) in this order: `$ARGUMENTS`, the branch name (`headRefName` for a PR), the PR title and body, then the commit messages.
2. Normalize to uppercase (`abc-12` → `ABC-12`).
3. If more than one distinct key turns up, list them and ask whether the notes go on one ticket or on each of them. Don't pick one yourself.
4. If no key turns up, ask the user for the ticket. If they say there isn't one, switch to chat-only mode for this run. Never invent a key.
5. Once you have a key, fetch the ticket with `getJiraIssue` (`cloudId: <jira_cloud_id>`, fields `summary`, `description`, `status`, `issuetype`, `comment`). Use its description and acceptance criteria to decide what QA should check. Also look through its comments for an existing QA-notes link from an earlier run. If there is one, ask whether to update that doc instead of making a second one.

## 3. Draft the notes

Write for a QA tester who has not read the code. Use product language, not implementation detail. Every point must come from the diff, the commits, the PR or the ticket. Never describe behaviour the change doesn't contain. If something QA needs can't be worked out (which environment, test credentials, an unclear expected result), add it to Open questions instead of guessing.

Sections, in this order:

- **What changed**: 2–4 sentences on what the user will see differently and why. Link the ticket and the PR when you have them.
- **Setup**: anything that must be done before testing, e.g. environment, feature flags, migrations, seed data, config or env vars, the build or deploy to use, or a user role. Write "None" if nothing is needed.
- **Test cases**: one `- [ ]` checkbox per case, so QA can tick them off. Each case gives the steps and the expected result. Cover the main path first, then edge cases and error states the code actually handles (validation, empty states, permissions, limits). Where the ticket has acceptance criteria, write at least one case for each criterion and name the criterion it covers.
- **Regression risk**: existing features that share code with this change and should get a quick re-check, and why each one is at risk (e.g. "Checkout — shares the updated price formatter").
- **Out of scope**: things that changed but don't need manual QA (refactors, logging, test-only changes), so QA doesn't spend time on them.
- **Open questions**: what QA needs answered before or during testing. Leave this section out if there are none.

## 4. Publish the notes as a doc

Publish the notes as a Claude Doc. Before the first docs call, load the docs skill (`anthropic-skills:docs`) if one is listed, otherwise call the docs `guide`. Then follow its create-and-fill flow: create the skeleton with one pending block per section, open it, and fill the sections one at a time.

- Title: `QA notes — <KEY> — <ticket summary>`. With no ticket, use `QA notes — <branch or PR title>`.
- Byline: the as-of date and the user, as the docs guide describes.
- When updating an existing doc from step 2, item 5, edit that doc in place instead of creating a new one.

If the docs connector isn't available, publish the same content as an Artifact page instead. If neither is available, print the notes as Markdown in chat and say that no doc was created.

## 5. Attach it to the Jira ticket

Skip this step in chat-only mode.

1. Look up the QA person's Jira account with `lookupJiraAccountId`, using the email from `qa_contact`. Use only an account ID the lookup returns. If it finds nothing, or finds more than one match, write their name as plain text instead of mentioning them.
2. Post one comment with `addCommentToJiraIssue` (`cloudId: <jira_cloud_id>`, `issueIdOrKey: <KEY>`). It should mention the QA person, link the doc, and list the test case titles so the ticket shows what is covered:

   ```
   @<QA person> QA notes are ready: <doc link>
   Covers: <PR link, or branch vs reference branch, or commit range>
   Test cases:
   - <case 1 title>
   - <case 2 title>
   Open questions: <count, or "none">
   ```

   When updating an existing doc, post a short comment saying the notes were updated and what changed. Don't repost the full comment.
3. If `qa_ready_transition` is set, move the ticket to that status. Call `getTransitionsForJiraIssue`, find the entry whose `name` matches `qa_ready_transition` (case-insensitive), and pass its `id` to `transitionJiraIssue`. Never hardcode a transition ID. If there is no matching transition (the ticket is already past that status, or the workflow is different), say so and continue; the command still succeeded.

## 6. Give QA access and report

No tool can share a Claude Doc or Artifact. Only the user can, from the **Share** button at the top of the doc in claude.ai, where they add people from their organization to view, comment or edit. So the last step is always the user's.

Finish with:

- The doc link.
- The Jira ticket link and comment, and whether the ticket was moved and to which status. In chat-only mode, say the notes weren't attached to any ticket.
- The share reminder: "Open the doc → **Share** → add <qa_contact> with comment access, so they can tick off test cases and leave comments." If QA is outside the organization and Share doesn't offer that, suggest exporting the doc as PDF or Markdown and attaching the file to the ticket.
- Anything left open: unresolved open questions, a QA mention that fell back to plain text, or a missing transition.

**Chat-only mode** (no Jira for this repo, or the user said there's no ticket): run steps 1, 3 and 4, skip steps 2 and 5, and still give the doc link and share reminder. If `qa_contact` is set, still name them in the reminder.

## Pitfalls

- **Stale branches.** Always `git fetch` before diffing branches. A diff against a local reference branch that's behind the remote pulls in other people's changes.
- **Two-dot vs three-dot diff.** Use `git diff <ref>...<branch>` (three dots) so the notes cover only what this branch added, not what changed on the reference branch since it was cut.
- **Merge commits.** Use `--no-merges` when listing commits, so merges from the reference branch aren't described as new work.
- **Duplicate docs.** Re-runs on the same ticket should update the existing doc (step 2, item 5), not create a second one.
- **Don't change the ticket otherwise.** This command reads the code and the ticket, writes one doc and one comment, and does at most one transition. It never edits the ticket's fields, assignee or description.
