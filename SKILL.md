# Configuration (seed -- copied to SharePoint by /setup-copilot)

This file documents what the team has configured. The live copy lives at `_knowledge/config.md` in the shared SharePoint folder; this file in the plugin is the seed used when first installing.

## Jira board

- **Cloud ID:** `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- **Projects watched:** `LEGAL`, `AIRD`
- **To-Do status:** `To Do`
- **Default JQL:** `project in (LEGAL, AIRD) AND status = "To Do" ORDER BY created ASC`

## SharePoint knowledge root

Set during `/setup-copilot` -- written to each user's `${CLAUDE_PLUGIN_DATA}/sharepoint-config.json`.

Default proposed during setup: the Legal team's shared folder at `mypos0.sharepoint.com/sites/legal`.

## Schedule

- **Daily run:** weekdays at 08:00 Europe/Sofia (registered as `legal-copilot-board-sweep`)
- **Cron expression:** `TZ=Europe/Sofia 0 8 * * 1-5`

## Pilot users

| Name | Role | Status |
|---|---|---|
| Ivan Troyanov | Pilot lawyer | v0.1 pilot |

## Configuration history

| Date | Change | By |
|---|---|---|
| (this section is appended automatically by /setup-copilot when run) | | |
