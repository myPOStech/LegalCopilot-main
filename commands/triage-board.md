---
description: Sweep the Jira board -- triage every To-Do legal ticket end-to-end. Runs daily at 08:00 CET on weekdays.
argument-hint: (no args)
---

# /triage-board

Process every To-Do ticket on the legal board: dedupe, classify, draft, review, file, comment. Designed to run autonomously every weekday at 08:00 CET (registered by `/setup-copilot`), and on demand whenever you want a fresh sweep.

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- Default JQL: `project in (LEGAL, AIRD) AND status = "To Do" ORDER BY created ASC`

## Phase 0: Verify configuration

Read `${CLAUDE_PLUGIN_DATA}/sharepoint-config.json`. If missing, print:

```
Legal Copilot is not configured yet. Run /setup-copilot first.
```

and stop. Otherwise continue.

---

## Phase 1: Refresh shared knowledge

### 1a. Read shared knowledge from SharePoint

- `_knowledge/patterns.md`
- `_knowledge/ticket-log.md`
- `_knowledge/feedback-log.md`
- `_knowledge/sharepoint-map.md` -- filing rules

### 1b. Detect lawyer corrections from recently closed tickets

```
JQL: project in (LEGAL, AIRD) AND status = Done AND resolved >= -7d ORDER BY resolved DESC
```

For each closed ticket with an "AI Triage" comment:
1. Read all comments using `mcp__atlassian__getJiraIssue`.
2. Find the AI draft and any subsequent lawyer edits or the final response sent.
3. Diff the AI draft against what the lawyer actually sent.
4. If material changes (not just whitespace/formatting), append to `_knowledge/feedback-log.md`:

```
## {ticket_key} ({YYYY-MM-DD})
**Skill used:** {legal-triage-* skill}
**AI draft excerpt:** "..."
**Lawyer changed:** {description of change}
**Category:** {factual correction | tone | scope | jurisdiction-specific | other}
**Lesson:** {what the AI should learn}
```

### 1c. Detect patterns

Review `_knowledge/feedback-log.md`. If 3+ corrections share a category + matter type + jurisdiction, add to `_knowledge/patterns.md`:

```
## Pattern: {description}
**Source:** {tickets}
**Rule:** {actionable rule}
**Confidence:** {High | Medium | Low}
**Affects skill:** {legal-triage-* skill}
```

If 5+ unprocessed feedback items suggest a skill change, surface a recommendation in the run summary -- DO NOT auto-edit any skill.

---

## Phase 2: Scan the board

```
JQL: project in (LEGAL, AIRD) AND status = "To Do" ORDER BY created ASC
```

If no tickets, print "Board clear -- no To-Do tickets" and skip to Phase 4 (SLA watchdog).

---

## Phase 3: Process each ticket

For each To-Do ticket, follow the same logic as `/triage` (see `commands/triage.md` Steps 2-9):

1. Fetch ticket
2. Dedupe → merge duplicates / link relates / escalate conflicts
3. Classify with the matching legal-triage skill
4. Apply risk gates
5. Devil's advocate review (via `triage-reviewer` subagent)
6. File attachments to SharePoint via `sharepoint-filer` skill
7. Create Outlook draft in AI Drafts folder
8. Post triage comment to Jira
9. Update `_knowledge/ticket-log.md`

**Ticket disposition:**

| Result | Status |
|---|---|
| Successfully triaged, no risk flags, Devil's advocate `approve` | Stay in To-Do; lawyer needs to review and run `/reply-and-close` |
| Risk flags / `human_review_required` | Stay in To-Do, marked clearly in comment |
| Duplicate of open ticket | Transition to Done with merge comment |
| Conflict | Stay in To-Do; both tickets flagged |
| Doesn't fit any of the 10 matter types | Stay in To-Do; clarification comment posted |

**The board sweep never auto-closes tickets that needed triage** -- closing is always a lawyer action via `/reply-and-close`. The only auto-close is for confirmed duplicates.

---

## Phase 4: SLA watchdog

```
JQL: project in (LEGAL, AIRD) AND statusCategory != Done ORDER BY created ASC
```

For each open ticket, calculate days since creation vs. the SLA for its priority. If >75% of SLA elapsed and no lawyer response since the AI triage comment, post:

```markdown
**SLA Warning**

This ticket was created {N} days ago. SLA for {priority} priority is {sla_days} days.
{remaining_days} day(s) remaining before SLA breach.

*Please review the AI draft (Outlook AI Drafts folder) and run `/reply-and-close` or escalate.*
```

---

## Phase 5: Print summary

```
Board sweep complete ({timestamp})

Processed: {total} tickets
  Triaged + draft ready: {count}
  Awaiting lawyer review: {count}
  Merged duplicates: {count}
  Conflicts escalated: {count}
  Clarification requested: {count}
  SLA warnings posted: {count}

{if pattern proposals: "PROPOSED PATTERNS (review and approve):"}
{pattern proposals}

Next scheduled run: {next weekday 08:00 CET}
```

---

## Hard rules

- NEVER send emails -- only create drafts in AI Drafts folder.
- NEVER auto-close a ticket that was actually triaged. Closing is lawyer-gated via `/reply-and-close`. Only confirmed duplicates auto-close.
- NEVER auto-edit any skill. Propose patterns, wait for human approval.
- NEVER leave a ticket without action -- every To-Do gets either a triage comment, a merge, a conflict flag, or a clarification request.
- NEVER override risk gates.
- Be conservative -- when uncertain, set `human_review_required = true`.
