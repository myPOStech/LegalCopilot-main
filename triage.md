---
description: Send the lawyer-reviewed AI draft, file the final email + documents to SharePoint, and transition the ticket to Done.
argument-hint: <TICKET-KEY> [--dry-run]
---

# /reply-and-close

After the lawyer has reviewed (and optionally edited) the AI draft in their Outlook AI Drafts folder, this command:

1. Sends the draft (now reviewed and edited by the lawyer)
2. Files the sent email + any documents to the SharePoint matter folder
3. Captures any lawyer edits as feedback for the knowledge base
4. Transitions the Jira ticket to Done

**This is the only command that sends emails. Every other command stops at "draft created."**

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`

## Input

`$ARGUMENTS`:
- `<TICKET-KEY>` (required) -- e.g., `LEGAL-4321`
- `--dry-run` (optional) -- show what would happen without sending or transitioning

---

## Step 1: Verify configuration

Read `${CLAUDE_PLUGIN_DATA}/sharepoint-config.json`. If missing, stop with `Run /setup-copilot first.`

---

## Step 2: Find the AI draft in Outlook

Search the user's Outlook `AI Drafts` folder for a draft tagged with the ticket key. The `/triage` command tags drafts with a custom property `x-mypos-legal-ticket: <TICKET-KEY>`.

`mcp__microsoft-365__outlook_email_search` filtered to:
- folder: `AI Drafts`
- custom property match: `x-mypos-legal-ticket eq '<TICKET-KEY>'`

If no draft is found, ask the user:

```
No AI draft found for {ticket_key} in your AI Drafts folder.

Possible reasons:
  (a) The ticket hasn't been triaged yet -- run /triage {key} first.
  (b) You moved or renamed the draft -- paste the message ID or paste the draft content here.
  (c) Another lawyer triaged it on their machine -- the draft is in their AI Drafts, not yours.
```

If multiple drafts found (e.g., re-triaged multiple times), pick the most recent and warn the user.

---

## Step 3: Read the current draft

Get the draft body, subject, and recipient list. Compare against what `/triage` originally produced (parse the AI Triage Jira comment for the original draft text).

**Detect lawyer edits:**

| Comparison | Disposition |
|---|---|
| Identical | No edits captured. Continue. |
| Whitespace/formatting only | No edits captured. Continue. |
| Material changes | Capture as feedback in Step 7 below. |

---

## Step 4: Confirm before sending

Show the lawyer:

```
About to send this email and close {ticket_key}.

To: {recipient list}
Subject: {final subject}
Body preview:
  {first 300 chars of body}

Attachments: {count} files
Material edits since AI draft: {yes/no}

Confirm? (y / n / show-full-body)
```

If `--dry-run`, print the above and stop without sending. Otherwise wait for explicit `y`.

---

## Step 5: Send the email

`mcp__microsoft-365__outlook_email_send` -- sends the draft, removes it from AI Drafts, places the sent copy in the user's Sent Items.

Capture the sent message ID for filing.

---

## Step 6: File the final email + documents to SharePoint

Invoke the `sharepoint-filer` skill (see `skills/sharepoint-filer/SKILL.md`) with:
- `ticket_key`
- `ticket_summary` (for the subfolder)
- `matter_type` (from the AI Triage Jira comment)
- `final_email_eml` (download the sent message as `.eml` via M365 MCP, save with `_final.eml` suffix)
- `attachments` (anything new since `/triage` ran)

The filer places the final `.eml` next to the original draft + Devil's advocate review files, with the suffix `_final.eml` so the chronology is obvious in the folder listing.

---

## Step 7: Capture lawyer edits as feedback

If material edits were detected in Step 3, append to `_knowledge/feedback-log.md`:

```markdown
## {ticket_key} ({YYYY-MM-DD})
**Skill used:** {legal-triage-* skill, from AI Triage comment}
**AI draft excerpt:** "{first material-changed sentence as written by AI}"
**Lawyer changed to:** "{first material-changed sentence as sent}"
**Category:** {best guess: factual correction | tone | scope | jurisdiction-specific | other}
**Captured by:** /reply-and-close
```

This is what makes the Copilot learn -- the gap between AI draft and what the lawyer actually sent is the highest-signal training data we have.

---

## Step 8: Update Jira

Post a final comment on the ticket:

```markdown
**Reply sent and ticket closed**

Sent to: {recipient list}
SharePoint folder: [{matter_type}/{ticket_subfolder}/]({folder_url})
Final email: [{filename}]({eml_url})

{if material edits: "Lawyer-edited draft -- {summary of edits} -- captured to feedback log."}

*Closed by Legal Copilot.*
```

Then transition to Done:
1. `mcp__atlassian__getTransitionsForJiraIssue` to find the Done transition ID.
2. `mcp__atlassian__transitionJiraIssue` to apply it.

---

## Step 9: Update the ticket log

Append to `_knowledge/ticket-log.md`:

```
| {YYYY-MM-DD} | {ticket_key} | {summary} | {type} | {priority} | Replied + closed | {edits captured: yes/no} |
```

Or update the existing row if one was added during `/triage`, replacing "Pending lawyer review" with "Closed".

---

## Step 10: Summarise

```
Replied and closed {ticket_key}:
  Sent: ✓ to {recipient list}
  SharePoint: filed (final email + {N} attachments)
  Jira: transitioned to Done
  Knowledge: ticket log updated
  {if edits captured: "Lawyer edits captured ({category}) -- will improve future drafts"}
```

---

## Hard rules

- NEVER send without explicit `y` confirmation in Step 4 (unless `--dry-run`, which doesn't send).
- NEVER bypass the Devil's advocate review -- if the original `/triage` flagged `human_review_required = true`, refuse to send and ask the user to confirm they've reviewed the flag.
- NEVER close a ticket where the AI Triage comment says `risk gates triggered` unless the lawyer has explicitly added a comment acknowledging the risk and approving the send. Look for a comment with text like `approved for send` or `risk reviewed` from the lawyer in the last hour.
- NEVER auto-resolve risk-flagged tickets even with confirmation -- always require an explicit lawyer comment in Jira saying the risk has been reviewed.
- ALWAYS file the final email to SharePoint before transitioning to Done. If filing fails, do not transition -- the audit trail must be complete.
- NEVER fabricate the recipient list -- it must come from the actual draft.
