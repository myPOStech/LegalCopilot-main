---
name: sharepoint-filer
description: File legal matter documents (drafts, Devil's advocate reviews, attachments, sent emails) to the team's shared SharePoint folder using the team's filing convention. Reads filing rules from `_knowledge/sharepoint-map.md` so lawyers can change folder names and naming templates without code changes. Invoked by `/triage`, `/file-to-sharepoint`, and `/reply-and-close`.
---

# sharepoint-filer

Files documents into the team's shared SharePoint folder using the convention defined in `_knowledge/sharepoint-map.md`. The filer is the single source of truth for "where does a file go and what is it named" -- every command that puts something in SharePoint goes through this skill.

## Inputs

The calling command passes:

- `ticket_key` (required) -- e.g., `LEGAL-4321`
- `ticket_summary` (required) -- the Jira ticket summary (used for the subfolder name)
- `matter_type` (required) -- one of the 10 matter types (NDA, contract review, regulatory question, corporate change, project, KYC support, GTCs, materials review, claims, inspection support)
- `attachments` (optional) -- list of `{name, url}` for files to download from Jira and upload to SharePoint
- `draft_text` (optional) -- the AI draft response, in markdown; saved as `.docx`
- `review_text` (optional) -- the Devil's advocate review, in markdown; saved as `.md`
- `final_email_eml` (optional) -- the sent message, downloaded as `.eml`; saved with the `_final.eml` suffix
- `comments_thread` (optional) -- the Jira comment thread as markdown; saved as `comments.md` (only when caller passes it)

## Step 1: Load filing rules

Read `_knowledge/sharepoint-map.md` from the configured shared SharePoint root (path in `${CLAUDE_PLUGIN_DATA}/sharepoint-config.json`). The map defines:

- The matter-type → top-level folder mapping
- The subfolder pattern (default: sanitised ticket summary, truncated to 80 chars)
- The filename template
- Per-matter-type overrides (if any)

If the map file is missing, fall back to the seed at `${CLAUDE_PLUGIN_ROOT}/knowledge/sharepoint-map.md` and warn the caller.

## Step 2: Resolve the destination folder

1. Look up `matter_type` in the map. If not found, abort and return an error -- never invent a folder.
2. Compute the ticket subfolder:
   - Default pattern: take `ticket_summary`, replace `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|` with `-`, collapse runs of whitespace and dashes, trim leading/trailing punctuation, truncate to 80 chars.
   - If the result is empty, fall back to `ticket_key`.
3. If a per-matter-type override is active in the map, apply it (e.g., `{counterparty}/{ticket_summary}` for NDAs).

## Step 3: Resolve folder collisions

Check whether the destination subfolder already exists via `mcp__microsoft-365__sharepoint_list_items`:

| State | Action |
|---|---|
| Doesn't exist | Create with `mcp__microsoft-365__sharepoint_create_folder` |
| Exists, contains files prefixed with this ticket key | Use it (this is the same matter coming back) |
| Exists, contains files prefixed with a different ticket key | Append ` (TICKET-KEY)` to the new folder name and create that |

## Step 4: Resolve filenames

Per the filename template in the map (default below):

```
{TICKET-KEY}_{counterparty-or-tag}_{YYYY-MM-DD}_{role-marker}.{ext}
```

| Role | Marker | Extension |
|---|---|---|
| Original draft | `v{N}` | `.docx` |
| Devil's advocate review | `v{N}_review` | `.md` |
| Attachment from Jira | `attachment_{original-stem}` | original |
| Final email (sent) | `final` | `.eml` |
| Comments thread | `comments` | `.md` |

`counterparty-or-tag` is derived from the ticket -- look for a company name in the description first; otherwise use a 3-word slug of the summary. `YYYY-MM-DD` is the current date. `v{N}` increments if a file with the same prefix already exists.

The `_review` suffix on the Devil's advocate file ensures alphabetical sort places it next to the draft it reviewed -- they MUST end up in the same folder.

## Step 5: Upload

For each file the caller provided:

- Convert `draft_text` from markdown to `.docx` before upload (use the docx skill if available).
- Save `review_text` as plain markdown.
- Download Jira attachments via the Atlassian MCP, then upload to SharePoint with the prefixed filename.
- Save the final email by downloading the sent message as `.eml` from Microsoft Graph.

Use `mcp__microsoft-365__sharepoint_upload_file` for each. Capture the SharePoint URL of every uploaded file.

## Step 6: Return

Return a JSON object the caller can use to build a Jira comment or chat summary:

```json
{
  "folder_url": "https://...",
  "files": [
    { "role": "draft", "name": "...", "url": "..." },
    { "role": "review", "name": "...", "url": "..." },
    { "role": "attachment", "name": "...", "url": "..." },
    { "role": "final_email", "name": "...", "url": "..." }
  ],
  "warnings": []
}
```

## Hard rules

- NEVER overwrite an existing file -- always increment `v{N}`.
- NEVER move files between folders without explicit caller instruction.
- NEVER strip the ticket key from filenames -- it is the cross-reference back to Jira.
- NEVER place Devil's advocate review files anywhere other than the same folder as the work they reviewed.
- NEVER invent a matter-type → folder mapping. If the map doesn't have it, abort and return an error.
- If any upload fails (auth, quota, network), report the failure clearly and DO NOT silently skip files.
