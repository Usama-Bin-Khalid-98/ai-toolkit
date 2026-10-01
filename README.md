# ai-toolkit

Reusable [Claude Code](https://claude.com/claude-code) slash commands for the everyday dev workflow: opening PRs, filing Jira tickets, writing QA notes and putting together release notes.

Each command is a single Markdown file under [.claude/commands/](.claude/commands/). Copy the ones you want into a project, and Claude Code picks them up as `/<command-name>`.

## Commands

| Command | What it does |
| --- | --- |
| [`/create-pr`](.claude/commands/create-pr.md) | Opens a PR for the current branch, or updates the existing one. It picks the base branch from your merge flow, puts the Jira key in the title so the PR shows on the ticket, requests the default reviewers and can move the ticket to "Code Review". If you're on a protected branch, it offers to move your changes to a new feature branch first. |
| [`/create-ticket`](.claude/commands/create-ticket.md) | Turns client evidence (screenshots, a pasted message, an email thread) into a Jira ticket. It works out the type, writes the title and description, checks for duplicates, masks personal data and shows you the draft before creating anything. |
| [`/qa-notes`](.claude/commands/qa-notes.md) | Writes QA notes for a PR, a branch or a range of commits and publishes them as a Claude Doc. It then comments on the Jira ticket, mentioning the QA contact, and can move the ticket to "Ready for QA". |
| [`/release-notes`](.claude/commands/release-notes.md) | Builds release notes from a Jira fix version. It checks them against the git diff between release tags so nothing that shipped is left out, then writes the result to Confluence, a changelog, or the chat. |
| [`/pr-status`](.claude/commands/pr-status.md) | Lists your open PRs and the PRs waiting on your review across all your GitHub repos, grouped by what needs doing next: failing CI, changes requested, conflicts, ready to merge, waiting on others. It flags stale PRs and PRs with no reviewers, and can show each ticket's Jira status. It doesn't depend on any project, so install it once in `~/.claude/commands/`. It only reads, and changes nothing unless you agree to a follow-up. |

## Setup

1. Copy the command files into your project's `.claude/commands/` directory, or into `~/.claude/commands/` to use them in every project:

   ```sh
   cp .claude/commands/*.md /path/to/your-project/.claude/commands/
   ```

2. Run a command. The first time, it asks for any settings it's missing (see below) and saves your answers into its own file, so later runs don't ask again.

### Configuration

There's no separate config file. Each command has a **Data required** section at the top with its settings:

```md
- `jira_project_key`: *(not set — Jira project prefix, e.g. ABC for ABC-1234, used to detect ticket keys)*
```

You can fill these in by hand, or let the command fill them in on its first run. Settings marked *optional* can be left unset. Since the answers are saved into the command file, commit it once it's set up so your team uses the same settings.

Settings that most commands use:

- `jira_base_url` / `jira_site_base`: your Atlassian site, e.g. `https://your-org.atlassian.net`
- `jira_cloud_id`: your Atlassian cloud ID, used by the Atlassian MCP tools
- `jira_project_key`: your Jira project prefix, e.g. `ABC`

Each command file lists its other settings (merge flow, reviewers, QA contact, Confluence space, tag pattern and so on).

### Requirements

| Needed for | Requirement |
| --- | --- |
| All commands | [Claude Code](https://docs.claude.com/en/docs/claude-code) |
| `/create-pr`, `/qa-notes` (PR input), `/pr-status` | [GitHub CLI](https://cli.github.com/), signed in with `gh auth login`, and an `origin` remote |
| Jira and Confluence steps | The Atlassian connector in Claude Code |
| `/qa-notes` doc output | The Claude Docs connector. Without it, the notes are published as an Artifact page or printed in chat. |

Every command still works without Jira, in a reduced mode: `/create-ticket` prints a draft ticket in chat, `/qa-notes` publishes the doc without attaching it to a ticket, `/release-notes` builds the notes from git alone, `/create-pr` skips the ticket link and status change, and `/pr-status` shows ticket keys without Jira statuses.

## Usage examples

```text
/create-pr
/create-pr this is a hotfix
/create-ticket <paste the client's email or attach a screenshot>
/qa-notes #123
/qa-notes feature/abc-12-login against stage
/release-notes 5.4.8
/pr-status
/pr-status reviews org:your-org
/pr-status mine here
```

## Adding a command

Add a Markdown file to `.claude/commands/` with a `description` in its front matter. Write it the same way as the existing ones:

- Put the settings in a **Data required** section, plus a **Step 0** that asks for any missing values and saves the answers into the file.
- Ask for confirmation before anything other people will see (creating tickets, force-pushing, changing a ticket's status).
- Offer a reduced mode for when a tool or connector isn't available, instead of stopping.
- End with a **Pitfalls** section listing mistakes to avoid.
