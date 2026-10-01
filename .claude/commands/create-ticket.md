---
description: Turn client evidence (a screenshot, pasted message or email thread) into a well-formed Jira ticket with a proper title, description and type
---

Create a Jira ticket from evidence a client or teammate shared. The evidence and any extra context from the user, if any: $ARGUMENTS

## Data required

This repo's ticket defaults, kept inline so this command doesn't need a separate config file. Edit the values below directly, or let the command prompt you and fill them in itself the first time it runs.

- `jira_base_url`: *(not set — Atlassian site URL, used to build ticket links, e.g. https://your-org.atlassian.net)*
- `jira_cloud_id`: *(not set — Atlassian cloud ID, needed for every Jira call via MCP)*
- `jira_project_key`: *(not set — Jira project new tickets go into, e.g. ABC)*
- `default_issue_type`: *(not set, optional — issue type used when the evidence doesn't clearly point to a bug or a feature, e.g. Task)*
- `default_labels`: *(not set, optional — labels added to every ticket this command creates, e.g. client-reported)*
- `default_assignee`: *(not set, optional — person new tickets are assigned to, as "Name <email>"; omit to leave tickets unassigned)*

**Step 0 — Resolve the data above** (do this before anything else): for any value still marked *(not set — ...)*, ask the user for it now — skip the ones marked optional if they have nothing to give. Once they answer, edit this file (`create-ticket.md`) and replace each bullet above with the value they gave, so future runs don't ask again. The three Jira fields are required: without them no ticket can be created, so if the user can't give them, offer to draft the ticket in chat only (see step 6) instead of blocking.

## 1. Collect the evidence

The evidence comes in `$ARGUMENTS`, in the conversation, or both. It can be any mix of:

- **Screenshots or images**: a file path or an image pasted into the chat. Open each one with the Read tool and read everything on it: error messages, URLs, page titles, visible data, timestamps, browser or device details.
- **Pasted text**: a Slack or Teams message, a support chat, a client note.
- **An email thread**: read it oldest to newest. Later replies often change or narrow the original ask, and the latest agreed version is what goes in the ticket. Note who asked for what.
- **Files or links**: a local file path (read it), or a Jira or Confluence link (fetch it with the Atlassian tools, e.g. as a related or parent ticket).

If no evidence was given at all, ask the user to paste or attach it and stop. Don't write a ticket from the user's one-line summary alone unless they say that's all there is.

## 2. Understand the ask

Work out, from the evidence only:

- **Type**: a **Bug** (something that used to work, or should work, is broken), a **Story** or feature (new capability), or a **Task** (a change, config, data fix or chore). If it's unclear, use `default_issue_type`, or ask.
- **What is affected**: the product area, page, flow or feature, in the product's own terms.
- **For a bug**: steps to reproduce, expected result, actual result, how often it happens, and the environment (prod/staging, browser, device, app version, account or record IDs) where the evidence shows it.
- **For a feature or task**: what the client wants, why they want it, and what "done" looks like.
- **Who reported it and when**: the client or person, and the date of the message.
- **Urgency signals**: deadlines, "blocking", "all users", revenue or data loss. Report these as evidence, not as a priority you decided (see step 4).

If the evidence contains more than one separate request (common in email threads), list them and ask whether to create one ticket per request or a single ticket. Don't merge unrelated asks into one ticket.

## 3. Check for duplicates

Before drafting, search for an existing ticket covering the same thing:

```
searchJiraIssuesUsingJql
  cloudId: <jira_cloud_id>
  jql:     project = <jira_project_key> AND statusCategory != Done AND text ~ "<2–4 distinctive keywords>" ORDER BY updated DESC
  fields:  ["key","summary","status","issuetype"]
  maxResults: 10
```

Try two or three keyword sets (the error message, the feature name, the page). If a likely match turns up, show it to the user and ask whether to create a new ticket, add the evidence as a comment on the existing one (`addCommentToJiraIssue`), or create a new ticket and link it (`createIssueLink`). Don't decide this yourself.

## 4. Draft the ticket

Write for a developer who hasn't seen the evidence. Every statement must come from the evidence. Never invent steps, environments, error text or acceptance criteria. Anything a developer would need but the evidence doesn't give goes in **Open questions**.

**Title**: under 80 characters, specific and searchable. Name the affected area and the problem or outcome, e.g. `Checkout: discount code rejected for valid codes on mobile` or `Allow admins to resend user invites`. No client names, no "URGENT", no "Issue with…" or "Bug:" prefix (the issue type already says that).

**Description** (Markdown), sections in this order:

For a **Bug**:

- **Summary**: 1–2 sentences on what's broken and who it affects.
- **Steps to reproduce**: numbered steps. Only include steps the evidence supports. If they can't be worked out, say so here and add an open question.
- **Expected result** / **Actual result**: one line each. Quote error messages exactly.
- **Environment**: environment, browser/device, app version, affected account or record IDs, as far as the evidence shows.
- **Impact**: how many users, how often, workaround if any, as reported.
- **Source**: who reported it, how (email, Slack, screenshot) and when, with the relevant part of the message quoted verbatim.
- **Open questions**: leave out if there are none.

For a **Story** or **Task**:

- **Summary**: 1–2 sentences on what's being asked for.
- **Background**: why the client wants it, in their words where possible.
- **Requirements**: bullet list of what must be true.
- **Acceptance criteria**: `- [ ]` checklist, each item testable. Only criteria the evidence supports.
- **Source**: as above.
- **Open questions**: leave out if there are none.

**Sensitive data**: strip or mask personal data (client email addresses, phone numbers, passwords, tokens, card numbers, home addresses) from the description and quotes. Keep record IDs and account names only if they're needed to reproduce the issue. Mention in the final report what was removed.

**Priority**: leave it at the project default. Put the urgency signals from step 2 in the description (e.g. "Client says this blocks their launch on 12 Oct") and let the team set the priority.

## 5. Confirm, then create

Creating a ticket is visible to the whole team, so show the draft first: type, title, labels, assignee, and the full description. Ask the user to confirm or edit it. Apply their changes and only create the ticket once they say go.

Before creating, check the issue type's required fields:

1. `getJiraProjectIssueTypesMetadata` (`cloudId`, `projectIdOrKey: <jira_project_key>`) to get the issue type ID for the chosen type. If the type doesn't exist in this project, show the available ones and ask.
2. `getJiraIssueTypeMetaWithFields` for that type. If it has required fields beyond summary and description (e.g. component, sprint, a custom field), ask the user for their values. Never fill a required field with a guess.

Then:

- If `default_assignee` is set, look up their account with `lookupJiraAccountId` using the email. Use only an account ID the lookup returns. If it finds nothing or more than one match, leave the ticket unassigned and say so.
- Create the ticket with `createJiraIssue`: `cloudId`, `projectKey: <jira_project_key>`, `issueTypeName`, `summary` (the title), `description` (the Markdown description) with `contentFormat: "markdown"`, `assignee_account_id` if you have one, and `additional_fields` for `labels` (from `default_labels`, if set) and any required fields from above.
- If the user asked to put it under an epic or parent, pass `parent: <KEY>` when creating. If they asked to link it to another ticket (from step 3 or the evidence), call `createIssueLink` after the ticket exists.

The Atlassian tools can't upload attachments. If the evidence included screenshots or files, tell the user to drag them onto the ticket in Jira, and list which ones.

## 6. Report

Finish with:

- The ticket link: `<jira_base_url>/browse/<KEY>`, with its type and title.
- Attachments the user still needs to add by hand (screenshots, files).
- Any sensitive data that was removed from the description.
- Open questions left in the ticket, so the user can chase them with the client.
- Anything that didn't work: an assignee lookup that failed, a link that wasn't made.

**Chat-only mode** (no Jira configured, Atlassian tools not connected, or the user asked for a draft only): run steps 1, 2 and 4, skip 3 and 5, and print the title and description as Markdown ready to paste into Jira. Say clearly that no ticket was created.

## Pitfalls

- **Inventing detail.** Client evidence is often thin. A short ticket with clear open questions is better than a full one with made-up steps or acceptance criteria.
- **Reading only the first email.** The ask often changes further down the thread. Use the latest agreed version, and quote the part that states it.
- **Ticket titles copied from the client.** "Site broken!!" or "Question about invoices" can't be searched or triaged. Rewrite the title from what the evidence actually shows.
- **Several requests in one ticket.** Split them (step 2) unless the user says otherwise.
- **Duplicates.** Always search first (step 3). A comment on an existing ticket is often the better outcome.
- **Personal data in Jira.** Tickets are widely visible and kept forever. Mask personal data before creating, not after.
- **Creating without confirmation.** Never call `createJiraIssue` before the user has seen and approved the draft.
