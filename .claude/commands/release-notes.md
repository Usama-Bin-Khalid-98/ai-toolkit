---
description: Generate a release notes page from a Jira fix version, cross-checked against the git tag diff so nothing shipped gets left out
---

Generate release notes for a release. The version, if named, is in `$ARGUMENTS` (a fix version name like `5.4.8` or `Release version 5.4.8`); otherwise use the most recently released version.

## Data required

This repo's release-notes defaults, kept inline so this command doesn't need a separate config file. Edit the values below directly, or let the command prompt you and fill them in itself the first time it runs.

- `jira_cloud_id`: *(not set — Atlassian cloud id)*
- `jira_project_key`: *(not set — Jira project prefix, e.g. ABC)*
- `jira_site_base`: *(not set — Atlassian site URL, e.g. https://your-org.atlassian.net)*
- `fix_version_prefix`: *(not set — literal text before the version number in fix version names, e.g. "Release version " for "Release version 5.4.8", or "" if versions are named bare like 5.4.8)*
- `confluence_space_key`: *(not set — required only if output_target is confluence; the space key used in CQL search, e.g. ABC)*
- `confluence_space_id`: *(not set — required only if output_target is confluence; the numeric space id used to create the page)*
- `confluence_parent_page_id`: *(not set — required only if output_target is confluence; the page always goes into this one space, under this one parent)*
- `contributors`: *(not set, optional — people always credited on every release page, e.g. a release manager, as "Name (account_id)"; omit to only credit assignees found on the issues themselves)*
- `tag_pattern`: *(not set, optional — glob `git tag -l` uses to find this repo's release tags for the §2.5 cross-check; omit to skip that cross-check entirely)*
- `output_target`: *(not set — confluence / changelog / print)*

**Step 0 — Resolve the data above.** For any value still marked *(not set — ...)*, ask the user for it now — skip the ones marked optional, and skip the three Confluence fields if `output_target` isn't `confluence`. At minimum, `jira_project_key` and `jira_cloud_id` are needed to look anything up at all; if the user has neither and doesn't use Jira for this repo, offer to fall back to git-only mode (see the note at the end of §2.5) instead of blocking. Once the user answers, edit this file (`release-notes.md`) and replace each bullet above with the value they gave, so future runs don't ask again.

## 1. Identify the release

If `$ARGUMENTS` names a version, use it. Otherwise find the most recently released one:

```
searchJiraIssuesUsingJql
  jql:    project = <jira_project_key> AND fixVersion in releasedVersions() ORDER BY updated DESC
  fields: ["key","fixVersions"]
  maxResults: 5
```

The `fixVersions` object carries `name`, `id`, `releaseDate`, `released`, and often a `description` with the release highlights — use that description as the first source for the Description cell.

If `output_target` is `confluence`, check whether a page already exists so a release is never written up twice:

```
searchConfluenceUsingCql
  cql: space = <confluence_space_key> AND type = page AND title ~ "<version name>"
```

If one exists, say so and ask whether to update it (`updateConfluencePage`) instead of creating a duplicate. If several versions are missing notes, confirm which to do before writing pages in bulk.

## 2. Pull the issues

Every issue carrying the fix version goes in the table, whatever its status:

```
searchJiraIssuesUsingJql
  jql:    project = <jira_project_key> AND fixVersion = "<fix_version_prefix><X.Y.Z>" ORDER BY key DESC
  fields: ["key","summary","issuetype","status","assignee"]
  maxResults: 100
  responseContentFormat: markdown
```

Keep `fields` to that list — asking for descriptions or `*all` blows past the tool's output limit. If the result is saved to a file, read it with `jq` rather than dumping it. If `pageInfo.hasNextPage` is true, follow `endCursor` via `nextPageToken` until every issue is collected.

Split issues as you read them: **Bug**-type feeds Bug Fixes, Task/Story feeds New Features and Improvements. Note any release-marker Epic (e.g. titled with the release date) as a sanity check on the date; it still gets a table row but never appears in the written sections.

## 2.5 Cross-check against git (catch what Jira missed)

This is the step that keeps Jira from silently under-reporting a release. Skip it entirely if `tag_pattern` isn't set in the config.

1. Find the git tag for this version: `git tag -l "<tag_pattern>"`, matched against the version name (try the bare number, with a leading `v`, and any project-specific tag convention).
2. Find the tag immediately before it in `git tag --sort=-creatordate`.
3. `git log <previous_tag>..<target_tag> --no-merges --pretty=format:"%H%x09%s"` (or `gh pr list --state merged --search "merged:<date range>"` if a GitHub remote is available and PR titles carry more signal than commit subjects).
4. For each commit/PR, look for a `<jira_project_key>-<number>` key in the subject. Compare that set of keys against the fix-version issue list pulled in §2:
   - **Referenced in git but not on the fix version** — the ticket exists but wasn't tagged with this fixVersion, or was pushed without a ticket key at all. Flag it; ask whether to fold it into the notes (as its own line, sourced from the commit/PR — not fabricated) or leave it out because it was actually cherry-picked in from unrelated work.
   - **On the fix version but never appears in the git range** — likely not code (e.g. a config/content change, or a ticket attached to the wrong fixVersion). Don't drop it from the table, but don't force it into a written section either if there's no evidence of what shipped.
5. If no `gh`/repo access at all, skip this step and say so in the final report rather than guessing.

**Git-only mode** (no Jira for this repo, chosen in Step 0): skip §1's Jira lookups and §2 entirely. Find the version's tag and the previous tag as in steps 1–2 (use the most recent tag if `$ARGUMENTS` names none), then build the issue list from step 3's commits/PRs instead — one row per commit/PR, using its subject as the summary and its tag date as the release date. Classify each line as a fix, feature or improvement by what the change actually did. Leave out the Jira-only pieces (version link, issue-type column, assignee mentions), and say in the final report that the notes came from git alone.

## 3. Draft the narrative

Four things must be written, not copied:

**Description cell** — 3–6 numbered highlight lines, always ending with `Bug fixes.` Written in past tense, product language, one line per shipped capability. Source them from the Task/Story issues, the Jira version description, and anything surfaced in §2.5:

```
1: Added Discount codes functionality.
2: Improved Allocations UI.
3: Enabled admins to resend invite to users.
4: Improved map pin/logo quality on dashboard.
5: Bug fixes.
```

**Summary** — one paragraph, 2–3 sentences, opening "This release introduces…", closing with a nod to bug fixes and stability improvements.

**New Features** vs **Improvements to existing features** — split the highlights: brand-new capability ("Added…", "Enabled…") goes under New Features; refinement of something that already existed ("Improved…", "Updated…", "Changed…") goes under Improvements. One per line, no bullets — the template uses hard line breaks.

**Bug Fixes** — one line per Bug-type issue, flat list, newest issue key first, each line prefixed with its key:

```
ABC-2492 — Locations list no longer flickers between empty and populated on the event map.
ABC-2474 — A single booking now sends one confirmation email instead of two.
ABC-2429 — Refunds are processed as refunds rather than charged as a negative amount.
```

Rules for these lines:

- Say what is fixed from the user's point of view, present tense — not the ticket title, not the underlying code change. "Cancelling a reservation no longer returns a 500", not "Cancel info api returning 500 error."
- One line each. Skip internal detail (endpoint names, cache settings) unless it's the only way to say what changed.
- A Bug-type ticket that's really small feature/polish work belongs under New Features or Improvements instead — never listed twice. Every Bug-type issue appears exactly once across the three written sections.
- If a release has no Bug-type issues, write `No bug fixes in this release.` rather than dropping the section.

Use this product's own vocabulary (domain terms, not implementation words lifted from ticket titles). Never claim a feature or fix that isn't backed by an issue in the table or a commit surfaced in §2.5.

## 4. Build the output

**If `output_target` is `confluence`:** title, then body via `createConfluencePage` with `contentFormat: "html"`, `spaceId: <confluence_space_id>`, `parentId: <confluence_parent_page_id>`. Title shape — for a same-day write-up use the current local date/time, for a back-filled release use its actual release date so it sorts alongside its siblings:

```
Release Notes - <Product Name> - <version name> - Sep 03 16:00
```

```html
<p><strong>How to use this page:</strong></p>
<p>Find your selected Jira issues in the table below. Select the expand to use them as your source of truth to write release notes.</p>

<table>
  <tbody>
    <tr><th><p><strong>Release</strong></p></th>
        <td><p><a href="<jira_site_base>/projects/<jira_project_key>/versions/VERSION_ID" data-card-appearance="inline"><version name></a></p></td></tr>
    <tr><th><p><strong>Date</strong></p></th>
        <td><p><time datetime="YYYY-MM-DD"></time></p></td></tr>
    <tr><th><p><strong>Version</strong></p></th>
        <td><p><version name></p></td></tr>
    <tr><th><p><strong>Description</strong></p></th>
        <td><p>1: …<br />2: …<br />3: …<br />4: Bug fixes.</p></td></tr>
    <tr><th><p><strong>Contributors</strong></p></th>
        <td><p><!-- one <span data-type="mention" data-user-id="..."> per configured contributor, plus distinct assignees found on the issues --></p></td></tr>
  </tbody>
</table>

<p>Before you share the page, review the contents of each Jira issue and remove any sensitive data.</p>

<table>
  <thead>
    <tr><th><p><strong>Issue</strong></p></th><th><p><strong>Summary</strong></p></th><th><p><strong>Issue Type</strong></p></th><th><p><strong>Fix versions</strong></p></th></tr>
  </thead>
  <tbody>
    <tr><td><p><a href="<jira_site_base>/browse/ABC-1234" data-card-appearance="inline">ABC-1234</a></p></td>
        <td><p>Issue summary verbatim from Jira</p></td>
        <td><p>Bug</p></td>
        <td><p><code><version name></code></p></td></tr>
    <!-- one row per issue, newest key first -->
  </tbody>
</table>

<h1>Summary</h1>
<p>This release introduces …</p>

<h1>New Features</h1>
<p>Added …<br />Enabled …</p>

<h1>Improvements to existing features</h1>
<p>Improved …<br />Updated …</p>

<h1>Bug Fixes</h1>
<p>ABC-1234 — …<br />ABC-1230 — …<br />ABC-1228 — …</p>
```

Rules:

- The `Date` cell is the version's `releaseDate`, not today.
- `VERSION_ID` is the numeric `id` from the `fixVersions` object.
- Issue summaries are copied verbatim, HTML-escaped (`&amp;`, `&quot;`, `&#39;`).
- In Bug Fixes, Jira keys stay plain text — the issue table above already carries the links.
- Only use account ids that came from `contributors` in config or appeared in tool output on an actual assignee. Never guess an id — omit the mention rather than invent one.
- Build the HTML in a local file and pass its content rather than assembling a huge string inline; for 25+ issues, generate the rows with a short script from the JQL results so no summary gets mistyped.
- Do not write `<ac:structured-macro>` — this template round-trips through HTML+ with `data-type` attributes; Confluence converts the issue links into Jira macros itself on save.

**If `output_target` is `changelog`:** convert the same four written sections (Summary, New Features, Improvements, Bug Fixes) into a Markdown block and prepend it to `CHANGELOG.md` under its top-level heading, matching whatever section style the file already uses. Include the issue table as a compact list (`- [ABC-1234](link) — summary`) rather than an HTML table.

**If `output_target` is `print`:** print the four written sections as Markdown, plus the issue table, directly in the response — no file or page is created.

## 5. Verify before reporting

- If `output_target` is `confluence`: re-read the published page (`getConfluencePage`, `contentFormat: "markdown"`).
- Row count in the issue table == issue count from the JQL.
- Every Bug-type issue appears exactly once across Bug Fixes, New Features and Improvements — none dropped, none listed twice.
- Every highlight in Description traces to an issue in the table or a §2.5 finding.
- Date matches the version's release date; version name matches everywhere.
- Every §2.5 finding was either folded in or explicitly called out as excluded — not silently dropped.

Report the page URL / file path / printed notes, plus anything worth flagging — issues still sitting in Testing/Pre-Release, a version Jira hasn't marked released yet, or commits found in git that weren't reflected in Jira.

## Pitfalls

- **Duplicate pages.** Re-runs on the same version create repeated titles. Always search first (§1).
- **Unreleased version.** If `released` is false, the release may not have shipped; confirm before publishing rather than guessing.
- **Mislabelled tickets.** Plenty of issues typed `Bug` are actually small enhancements, and some typed `Task` are fixes. Place each line by what the ticket actually did, not by its issue type alone.
- **Tag/version name mismatches (§2.5).** A repo's git tags don't always match Jira version names literally (`v5.4.8` vs `5.4.8` vs `release-5.4.8`) — try the obvious variants before concluding there's no matching tag.
- **Annotated vs lightweight tags** can throw off `--sort=-creatordate` ordering — double-check the "previous" tag picked is really the immediately-preceding release.
- **Never mark the Jira version released**, transition issues, or edit tickets as part of writing notes. This command only reads Jira and git, and writes one output.
