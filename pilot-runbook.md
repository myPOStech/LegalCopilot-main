---
description: One-time setup -- configure SharePoint folder, create matter-type subfolders, register the daily 08:00 CET schedule.
argument-hint: (no args)
---

# /setup-copilot

Run this once after installing the plugin. It walks the lawyer through:

1. Confirming the shared SharePoint folder
2. Creating the 10 matter-type subfolders + `_knowledge/` brain folder
3. Seeding the knowledge files
4. Registering the weekday 08:00 CET scheduled run for `/triage-board`

Subsequent runs are idempotent -- if the structure already exists, it just verifies and reports.

---

## Step 1: Locate or ask for the shared SharePoint folder

Check `${CLAUDE_PLUGIN_DATA}/sharepoint-config.json`. If it exists, read the configured root and skip to Step 2.

If not, prompt the user:

```
Welcome to the myPOS Legal Copilot setup.

The Copilot needs a shared SharePoint folder where the team's brain lives
(patterns, ticket log, feedback) and where matter files are saved.

Default location:
https://mypos0.sharepoint.com/:f:/s/legal/IgD1ciPgZrRTQraRT4flBa4yAZC8NDvy0sv0YfcM6d3S_mw

Press Enter to use this, or paste a different SharePoint folder URL.
```

Validate the URL with `mcp__microsoft-365__sharepoint_list_items` -- the lawyer running setup must have read+write access. If access denied, instruct them to ask a folder owner to grant edit permission, then retry.

Save to `${CLAUDE_PLUGIN_DATA}/sharepoint-config.json`:

```json
{
  "root_url": "https://mypos0.sharepoint.com/sites/legal/Shared Documents/<folder>",
  "drive_id": "<resolved drive id>",
  "root_item_id": "<resolved item id>",
  "configured_by": "<user email>",
  "configured_at": "<ISO timestamp>",
  "version": "0.1.0"
}
```

---

## Step 2: Create matter-type subfolders

For each of the 10 matter types, ensure the folder exists (create with `mcp__microsoft-365__sharepoint_create_folder` if missing):

- `NDAs`
- `Contract Reviews`
- `Regulatory Questions`
- `Corporate Changes`
- `Projects`
- `KYC Support`
- `GTCs`
- `Materials Reviews`
- `Claims`
- `Inspection Support`
- `_knowledge`

Print a checklist showing created vs. already-existed.

---

## Step 3: Seed shared knowledge

For each seed file, check if the SharePoint copy exists. If not, upload from `${CLAUDE_PLUGIN_ROOT}/knowledge/`:

| Seed file | SharePoint destination |
|---|---|
| `knowledge/config.md` | `_knowledge/config.md` |
| `knowledge/patterns.md` | `_knowledge/patterns.md` |
| `knowledge/feedback-log.md` | `_knowledge/feedback-log.md` |
| `knowledge/ticket-log.md` | `_knowledge/ticket-log.md` |
| `knowledge/sharepoint-map.md` | `_knowledge/sharepoint-map.md` |

Never overwrite an existing SharePoint file -- if a file is already there, skip it and report "preserved existing".

---

## Step 4: Register the scheduled run

Use the `mcp__scheduled-tasks__create_scheduled_task` tool (or equivalent in the user's Claude Code installation) to register:

- **Name:** `legal-copilot-board-sweep`
- **Cron:** `0 6 * * 1-5` (06:00 UTC on Mon–Fri = 08:00 CET in winter)
  - Note: during summer (CEST = UTC+2), this fires at 09:00 local. If the team prefers strict 08:00 local year-round, switch to a timezone-aware schedule:
    `TZ=Europe/Sofia 0 8 * * 1-5`
- **Command:** `/triage-board`
- **Description:** `Daily legal board sweep at 08:00 Sofia time (weekdays).`

If a task with this name already exists, prompt the user to choose: keep existing, replace, or skip.

Print:

```
Scheduled run registered:
  Name: legal-copilot-board-sweep
  When: weekdays at 08:00 Europe/Sofia (CET / CEST)
  Action: /triage-board
  Manage: /schedule list  |  /schedule delete legal-copilot-board-sweep
```

---

## Step 5: Smoke test

Try the following and report success/failure for each:

1. **Atlassian MCP:** `mcp__atlassian__searchJiraIssuesUsingJql` with `project = LEGAL AND statusCategory != Done`, limit 1.
2. **Microsoft 365 / Outlook:** `mcp__microsoft-365__outlook_email_search` with `top: 1`.
3. **Microsoft 365 / SharePoint:** read `_knowledge/config.md` from the configured folder.
4. **Devil's advocate skill:** verify the skill is available (the `triage-reviewer` subagent will fail loudly if not).

Print a final readiness summary:

```
Setup complete.

✓ SharePoint root: {url}
✓ 10 matter-type folders created (or preserved)
✓ Knowledge files seeded
✓ Scheduled run: weekdays 08:00 Europe/Sofia
✓ Atlassian connection: OK
✓ Microsoft 365 connection: OK
✓ Devil's advocate skill: available

You can now run:
  /triage <TICKET-KEY>      -- triage one ticket
  /triage-inbox             -- triage unread emails
  /triage-board             -- sweep all To-Do tickets
  /file-to-sharepoint <KEY> -- file documents
  /reply-and-close <KEY>    -- send a reviewed draft

If anything failed above, share the error with the AI Transformation team.
```

---

## Hard rules

- NEVER overwrite an existing SharePoint file in Step 3 -- preserve team data.
- NEVER write the SharePoint config to a shared location -- it lives in `${CLAUDE_PLUGIN_DATA}` per-user.
- NEVER skip the smoke test -- if the lawyer hits a bad install, surface it now, not at 08:00 the next morning.
