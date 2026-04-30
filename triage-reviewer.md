# Changelog

All notable changes to the myPOS Legal Copilot plugin.

## [0.1.0] - 2026-04-27

Initial scaffold. Pilot release for Ivan Troyanov.

### Added
- Plugin manifest (`.claude-plugin/plugin.json`)
- MCP wiring for Atlassian (Jira + Confluence) and Microsoft 365 (Outlook + SharePoint)
- Five user-facing slash commands:
  - `/triage` -- triage a single Jira ticket or email
  - `/triage-board` -- sweep all To-Do tickets on the board
  - `/triage-inbox` -- sweep unread emails in the legal mailbox
  - `/file-to-sharepoint` -- file a ticket's attachments + draft into the right SharePoint folder
  - `/reply-and-close` -- send a lawyer-reviewed draft and close the ticket
  - `/setup-copilot` -- one-time setup (folder structure, scheduled run)
- SharePoint filer skill with team-editable folder mapping
- References to the 10 published legal-triage skills + Devil's advocate review
- Triage-reviewer subagent that runs Devil's advocate over every draft
- Shared knowledge files seeded for SharePoint bootstrap
- Daily 08:00 CET scheduled run (weekdays), registered by `/setup-copilot`

### Known gaps (tracked for v0.2)
- MCP server identifiers in `.mcp.json` may need adjustment depending on whether the team uses the Claude.ai-hosted or self-hosted Atlassian/Microsoft 365 MCP servers
- Folder collision detection currently appends ticket key on conflict; future version may offer interactive disambiguation
- Knowledge file conflict resolution (multiple lawyers writing simultaneously) is last-writer-wins; future version may add merge logic
