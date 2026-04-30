# myPOS Legal Copilot

AI triage and drafting assistant for the myPOS Legal team.

You give it a Jira ticket, an email, or point it at the board. It classifies the request, checks for duplicates, drafts a response in myPOS legal house style, runs a Devil's advocate review on the draft, files attachments to the right SharePoint folder, creates an Outlook draft for the lawyer to review, and updates Jira. It learns from lawyer feedback and gets better over time.

Runs entirely inside **Claude Code** -- no backend, no API keys, no server to maintain.

---

## Setup (one-time, ~5 minutes)

### Step 1: Install Claude Code

Download the desktop app from [claude.ai/download](https://claude.ai/download) and sign in with your myPOS account.

### Step 2: Install the Legal Copilot plugin

Open Claude Code and run, in this exact order:

```
/plugin marketplace add myPOSTech/LegalCopilot
/plugin install mypos-legal-copilot@mypos-legal
```

You'll be asked to confirm. Say yes.

### Step 3: Connect Jira and Outlook

The first time you run a command, Claude Code will ask for permission to connect to **Atlassian** (for Jira) and **Microsoft 365** (for Outlook + SharePoint). Click through the OAuth prompts -- one window each, log in with your myPOS account, done. You only do this once.

### Step 4: Configure your shared SharePoint folder

Run:

```
/setup-copilot
```

This will:
- Ask you to confirm the shared SharePoint folder path
- Create the 10 matter-type subfolders inside it (NDAs, Contract Reviews, Regulatory Questions, ...)
- Create the `_knowledge/` subfolder for the team's shared brain (patterns, ticket-log, feedback-log)
- Register the daily 08:00 CET scheduled run

You should only need to do this once for the whole team -- the first lawyer who installs runs it; everyone else inherits the configuration.

---

## What it can do for you

### Triage a single ticket on demand

```
/triage LEGAL-4321
```

Claude reads the ticket, classifies it, runs the matching legal-triage skill, drafts a response, has Devil's advocate review the draft, files any attachments to SharePoint, creates a draft reply in your Outlook AI Drafts folder, and posts the triage summary as a Jira comment. **The ticket does not move to Done unless the lawyer approves.**

### Review and send

After you've reviewed the AI draft in Outlook (and edited it if needed):

```
/reply-and-close LEGAL-4321
```

Sends your reviewed reply, transitions the ticket to Done, attaches the final email and any documents to the SharePoint folder, and logs the close to the team knowledge base.

### File documents to the right SharePoint folder

```
/file-to-sharepoint LEGAL-4321
```

Picks up Jira attachments + the latest draft and files them into the matter-type folder using the team's filing convention. Useful when a ticket has new documents added mid-flight.

### Triage the inbox

```
/triage-inbox
```

Scans your unread emails, dedupes against existing Jira tickets, creates new tickets for new matters, drafts replies in AI Drafts.

### Sweep the whole board

```
/triage-board
```

Processes every To-Do ticket on the board, end-to-end. Runs automatically every weekday at 08:00 CET once `/setup-copilot` has registered the schedule. Run it manually any time you want a fresh pass.

---

## How filing works

Files are saved into your shared Legal SharePoint folder, grouped by matter type:

```
<your shared SharePoint folder>/
  ├── NDAs/
  ├── Contract Reviews/
  ├── Regulatory Questions/
  ├── Corporate Changes/
  ├── Projects/
  ├── KYC Support/
  ├── GTCs/
  ├── Materials Reviews/
  ├── Claims/
  ├── Inspection Support/
  └── _knowledge/
```

Inside each folder, files for one ticket are grouped in a subfolder named after the ticket's title (sanitised to remove invalid filesystem characters and truncated to 80 chars). If two tickets have the exact same title the second one's folder gets the ticket key appended for disambiguation.

**File names** follow this pattern:

```
LEGAL-4321_AcmeCorp_2026-04-27_v1.docx        ← original draft
LEGAL-4321_AcmeCorp_2026-04-27_v1_review.md   ← Devil's advocate review
LEGAL-4321_AcmeCorp_2026-04-27_final.eml      ← the email actually sent
```

Devil's advocate reviews always sit next to the work they reviewed -- they go into the same skill folder as the original draft (e.g., an NDA review goes into `NDAs/`, not a separate review folder).

You can change the folder names or filing rules by editing one file: `_knowledge/sharepoint-map.md` in the shared folder. No developer needed.

---

## Safety rails (hard rules, never overridden)

- **Never sends emails** -- only creates drafts in your Outlook AI Drafts folder
- **Never closes a risk-flagged ticket** -- regulator, inspection, claim, or tight-deadline tickets always require a lawyer to approve before close
- **Never auto-edits its own rules** -- when it spots a recurring lawyer correction, it proposes a change and waits for sign-off
- **Never invents facts** -- if information is missing, it asks
- **Never names individuals** -- uses roles ("the Compliance team", not "Maria from Compliance")
- **Drafts are always marked** `[DRAFT - FOR LAWYER REVIEW BEFORE SENDING]`

---

## Updates

Run `/plugin update` whenever you want the latest version. New skills, fixes, and improved drafting style ship through the same channel. No reinstall needed.

If something looks broken after an update, you can pin to an earlier version or roll back -- ask Claude.

---

## Asking for help

Open Claude Code and say "ask the legal copilot maintainers" -- it'll prepare a bug report including your last command, the relevant Jira ticket, and the error, then post it to the maintainer channel. Or message the AI Transformation team directly.

## For pilot users

If you're the first lawyer installing this on a fresh machine, follow [docs/pilot-runbook.md](docs/pilot-runbook.md) — a 30-minute step-by-step guide that walks you through install, the OAuth dance, three smoke tests on a controlled test ticket, and verifying the next morning's automatic 08:00 sweep.

## Project structure (for the curious)

```
.claude-plugin/plugin.json   # plugin manifest
commands/                    # the slash commands you type
skills/                      # the legal triage playbooks + Devil's advocate + filer
agents/                      # the reviewer subagent
.mcp.json                    # connections to Jira + Microsoft 365
settings.json                # default permission allowlist
knowledge/                   # seed templates copied to SharePoint on first install
hooks/                       # post-edit hooks that capture lawyer corrections
```

The shared brain (patterns, ticket-log, feedback-log) lives on SharePoint, so every lawyer's copilot reads from and writes to the same source of truth.
