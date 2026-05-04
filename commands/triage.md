---
description: Triage a single Jira ticket or pasted email -- classify, dedupe, draft, review, file, post.
argument-hint: <TICKET-KEY | jira URL | pasted email>
---

# /triage

Triage a single legal request end-to-end. The ticket does **not** move to Done -- the lawyer reviews the draft in Outlook, then runs `/reply-and-close` to send and close.

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- Projects watched: `LEGAL`, `AIRD`

## Input

`$ARGUMENTS` is one of:
- A Jira ticket key (e.g., `LEGAL-4321`, `AIRD-44`)
- A Jira URL (extract the issue key from the path)
- Raw email content (subject + body + sender)

If `$ARGUMENTS` is empty, ask the user what to triage.

---

## Step 1: Load shared knowledge from SharePoint

Read these files from the team's shared SharePoint knowledge folder (path stored in `${CLAUDE_PLUGIN_DATA}/sharepoint-config.json`, written by `/setup-copilot`):

- `_knowledge/patterns.md` -- learned rules from lawyer feedback
- `_knowledge/ticket-log.md` -- recent tickets (last 30 days) for dedup and reference
- `_knowledge/feedback-log.md` -- correction history

Use `mcp__microsoft-365__sharepoint_read_file` for each. If any file is missing, fall back to the seed files in `${CLAUDE_PLUGIN_ROOT}/knowledge/` and tell the user "shared knowledge not yet bootstrapped -- run `/setup-copilot`".

---

## Step 2: Fetch the request content

**Jira key provided:** `mcp__atlassian__getJiraIssue` with `responseContentFormat: markdown`. Extract summary, description, reporter, attachments list, existing comments.

**Email pasted:** use the subject, body, and sender directly.

---

## Step 3: Deduplication check

JQL search for open near-duplicates:

```
project in (LEGAL, AIRD) AND statusCategory != Done AND summary ~ "{key terms}"
```

Also scan `_knowledge/ticket-log.md` for closed tickets with >90% semantic similarity.

**Decision tree:**

| Finding | Action |
|---|---|
| Duplicate of an open ticket | Link as "is duplicated by", copy content into the original as a comment, transition the new ticket to Done with comment "Merged into {key}". Stop. |
| >90% match to a closed ticket | Note the reference for Step 5 (adapt the previous response). Continue. |
| Conflict (same matter, contradictory asks) | Link both as "relates to", flag on both with conflict comment, escalate. Stop. |
| New request | Continue to Step 4. |

---

## Step 4: Classify using the matching legal-triage skill

Pick exactly one of the 10 published legal-triage skills based on the matter:

| Matter | Skill |
|---|---|
| NDA / confidentiality / CDA | `legal-triage-nda` |
| Third-party commercial contract | `legal-triage-contract-review` |
| Regulatory question / regulator request | `legal-triage-regulatory-question` |
| Entity restructuring, director, share | `legal-triage-corporate-change` |
| Multi-workstream legal involvement | `legal-triage-project` |
| KYC / corporate documentation about myPOS | `legal-triage-kyc` |
| Updates to myPOS T&Cs | `legal-triage-gtcs` |
| External-facing materials review | `legal-triage-materials-review` |
| Dispute / claim / litigation | `legal-triage-claims` |
| Regulatory inspection support | `legal-triage-inspection-support` |

If none of the 10 fit, post a clarification comment on the ticket asking what legal support is needed. Do NOT move to Done. Stop.

The chosen skill produces structured output (matter type, priority, SLA, jurisdiction, risk flags, missing-info questions, recommended action, and a draft response in myPOS legal house style).

**Risk gates (hard rules, never override):**
- Any risk flag (regulator / inspection / claim / tight deadline) → `human_review_required = true`
- Skill type is `claims` or `inspection_support` → `human_review_required = true`
- Confidence < 0.7 OR ambiguous classification → `human_review_required = true`

---

## Step 5: Devil's advocate review

Pass the draft from Step 4 to the `triage-reviewer` subagent (`agents/triage-reviewer.md`). The subagent runs the `devils-advocate-review` skill over the draft and returns:

- Issues found (each with severity: blocker / concern / nit)
- Suggested edits
- An overall verdict: `approve`, `revise`, or `escalate`

If verdict = `revise`: apply the suggested edits to the draft, then re-run review. Cap at 2 revision passes; if still `revise` after 2 passes, set `human_review_required = true` and surface the remaining issues to the lawyer.

If verdict = `escalate`: set `human_review_required = true` and include the escalation reasons in the Jira comment.

---

## Step 6: File attachments to SharePoint

Invoke the `sharepoint-filer` skill with:

- `ticket_key` (e.g., `LEGAL-4321`)
- `ticket_summary` (used for the subfolder name)
- `matter_type` (from Step 4 -- determines which top-level folder)
- `attachments` (list of attachment URLs from the Jira ticket)
- `draft_text` (from Step 5 -- saved as a `.docx`)
- `review_text` (from Step 5 -- saved as `..._review.md`)

The skill returns the SharePoint folder URL, which gets cited in the Jira comment.

---

## Step 7: Create the Outlook draft

`mcp__microsoft-365__outlook_email_create_draft` in the `AI Drafts` folder of the user's mailbox:

- To: original sender (or reporter email if from Jira)
- Subject: `Re: {original subject}`
- Body: the reviewed draft (HTML), with `[DRAFT - FOR LAWYER REVIEW BEFORE SENDING]` banner at top
- Custom property: `{"x-mypos-legal-ticket": "<ticket_key>"}` so `/reply-and-close` can find this draft later

---

## Step 8: Post the triage summary as a Jira comment

`mcp__atlassian__addCommentToJiraIssue` with `contentFormat: markdown`:

```markdown
---
## AI Triage

**Matter type:** {type}
**Priority:** {priority} | **SLA due:** {sla_days} calendar days
**AI confidence:** {confidence}%
**Jurisdictions:** {jurisdictions or "unspecified"}
**Skill used:** {legal-triage-* skill name}

{if risk flags: "**Risk flags:** {flags}"}
{if human_review_required: "**⚠ NEEDS HUMAN REVIEW BEFORE SEND**"}
{if similar past ticket: "**Similar past ticket:** {key} -- response adapted from previous matter"}
{if missing info: "**Missing information:**\n{questions}"}

**Recommended action:** {action}

**Devil's advocate review:** {verdict} -- {1-line summary of findings}

**Filed to SharePoint:** [{matter_type}/{ticket_summary_subfolder}]({sharepoint_url})

---
## Draft Response (for lawyer to review, edit, and send)

{reviewed draft text}

---
*Outlook draft saved to AI Drafts folder. Run `/reply-and-close {ticket_key}` after review.*
---
```

---

## Step 9: Update shared knowledge

Append a row to `_knowledge/ticket-log.md` on SharePoint:

```
| {YYYY-MM-DD} | {ticket_key} | {summary} | {type} | {priority} | Triaged + draft ready | Pending lawyer review |
```

Use `mcp__microsoft-365__sharepoint_upload_file` (or update-in-place) to write back. Last-writer-wins; the file is small enough that conflicts are rare.

---

## Step 10: Summarise to the user

```
Triage complete for {ticket_key}:
  Matter: {type} | Priority: {priority} | SLA: {sla_days}d
  Skill: {legal-triage-skill-name}
  Risk flags: {flags or "none"}
  Devil's advocate verdict: {approve | revise | escalate}
  Filed to: {sharepoint folder URL}
  Outlook draft: AI Drafts > "Re: {subject}"
  Jira comment posted.
  {if human_review_required: "⚠ Lawyer review required before /reply-and-close"}
```

---

## Hard rules

- NEVER send emails -- only create drafts.
- NEVER move the ticket to Done. That's `/reply-and-close`'s job, after lawyer review.
- NEVER fabricate facts -- if information is missing, ask via the Jira comment.
- NEVER mention individuals by name -- use roles ("Compliance team", not "Maria").
- NEVER override risk gates.
- Be conservative -- when uncertain, set `human_review_required = true`.
