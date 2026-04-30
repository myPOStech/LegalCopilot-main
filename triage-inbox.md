---
description: File a ticket's attachments and current draft into the right SharePoint matter-type folder.
argument-hint: <TICKET-KEY> [--include-comments]
---

# /file-to-sharepoint

Pick up the attachments + latest AI draft for a Jira ticket and file them into the SharePoint matter-type folder using the team's filing convention.

Useful when:
- A ticket has new documents added mid-flight that weren't there during initial triage
- You want to manually trigger filing without re-running the full `/triage` flow
- A lawyer wants to attach their own redlined version to the matter folder

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`

## Input

`$ARGUMENTS`:
- `<TICKET-KEY>` (required) -- e.g., `LEGAL-4321`
- `--include-comments` (optional) -- also save Jira comments as a `comments.md` file in the matter folder

## Step 1: Verify configuration

Read `${CLAUDE_PLUGIN_DATA}/sharepoint-config.json`. If missing, print:

```
Legal Copilot is not configured yet. Run /setup-copilot first.
```

and stop.

---

## Step 2: Fetch the ticket

`mcp__atlassian__getJiraIssue` for the given key with `responseContentFormat: markdown`. Extract:
- Summary (used for the ticket subfolder name)
- Description
- Attachments (URLs + filenames)
- All comments (only if `--include-comments`)
- Any AI Triage comment (parse the matter type out of it)

If the ticket doesn't have an AI Triage comment yet, ask the user:

```
This ticket hasn't been triaged yet. Should I:
  (a) Run /triage first, then file (recommended)
  (b) File anyway -- I'll ask you for the matter type
```

---

## Step 3: Resolve the destination folder

Read `_knowledge/sharepoint-map.md` (the team's filing rules) and the matter type from Step 2 to determine the top-level folder. Default mapping:

| Matter type | Top-level folder |
|---|---|
| NDA | `NDAs/` |
| Contract review | `Contract Reviews/` |
| Regulatory question | `Regulatory Questions/` |
| Corporate change | `Corporate Changes/` |
| Project | `Projects/` |
| KYC support | `KYC Support/` |
| GTCs | `GTCs/` |
| Materials review | `Materials Reviews/` |
| Claims | `Claims/` |
| Inspection support | `Inspection Support/` |

The team can override this in `_knowledge/sharepoint-map.md` -- read that file first and let it win.

---

## Step 4: Compute the ticket subfolder name

Per the team rule: **subfolder name = sanitised ticket summary, truncated to 80 chars**.

Sanitisation rules:
- Replace `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|` with `-`
- Collapse runs of whitespace and dashes
- Trim leading/trailing punctuation and whitespace
- Truncate to 80 characters
- If the result is empty (rare -- summary was all special chars), fall back to the ticket key

Check if the folder already exists in the matter-type top-level via `mcp__microsoft-365__sharepoint_list_items`:

| Folder state | Action |
|---|---|
| Doesn't exist | Create it with `mcp__microsoft-365__sharepoint_create_folder` |
| Exists, was created for THIS ticket (folder contains files prefixed with this ticket key) | Use it |
| Exists, was created for a different ticket (collision) | Append ` (TICKET-KEY)` to the new folder name and create that |

---

## Step 5: Resolve filenames

Per the team filing rule:

```
{TICKET-KEY}_{counterparty-or-subject-tag}_{YYYY-MM-DD}_v{N}.{ext}
```

Where:
- `counterparty-or-subject-tag` is derived from the ticket (look for company name in description; fall back to a 3-word slug of the summary)
- `YYYY-MM-DD` is the current date
- `v{N}` increments if a file with the same prefix already exists in the folder

Devil's advocate review files use the same prefix with `_review.md` suffix:

```
LEGAL-4321_AcmeCorp_2026-04-27_v1.docx          ← original draft
LEGAL-4321_AcmeCorp_2026-04-27_v1_review.md     ← Devil's advocate review (sits next to it)
```

This naming guarantees alphabetical sort puts the review next to its target.

---

## Step 6: Upload files

For each file to upload, use `mcp__microsoft-365__sharepoint_upload_file`:

1. **Jira attachments** -- download via Atlassian MCP, upload to the ticket subfolder, preserving original filename but prefixed with the ticket key + date
2. **AI draft** -- if a recent draft exists in the most recent AI Triage Jira comment, save as `.docx` (convert markdown to docx first)
3. **Devil's advocate review** -- if found in the AI Triage comment, save as `.md` with `_review.md` suffix
4. **Comments** (only if `--include-comments`) -- save the full comment thread as `comments.md`

For each file, capture the SharePoint URL.

---

## Step 7: Update the Jira ticket

Add a comment to the ticket:

```markdown
**Files filed to SharePoint**

- [{matter_type}/{ticket_subfolder}/]({sharepoint_folder_url})

Files:
- {file1_name} -- [open]({file1_url})
- {file2_name} -- [open]({file2_url})

*Filed by Legal Copilot.*
```

---

## Step 8: Summarise

```
Filed {N} files for {ticket_key} to:
  {sharepoint folder URL}

  Original draft: {filename}
  Devil's advocate review: {filename}
  Attachments: {count}
  {if --include-comments: "Comments: comments.md"}

Jira comment posted.
```

---

## Hard rules

- NEVER overwrite an existing file -- always increment `v{N}`.
- NEVER move files between folders without explicit user instruction. If a file is already filed elsewhere, report it and stop.
- NEVER strip the ticket key from filenames -- it's the cross-reference back to Jira.
- NEVER place Devil's advocate review files anywhere other than the same folder as the work they reviewed.
- If SharePoint upload fails (auth, quota, network), report the failure clearly and DO NOT silently skip files.
