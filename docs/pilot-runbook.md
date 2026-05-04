# Pilot runbook — myPOS Legal Copilot v0.1

A step-by-step guide for the first lawyer to install and stress-test the Legal Copilot. Designed for Ivan Troyanov (pilot user) but works for any future pilot.

This is a **runbook** — follow it top-to-bottom, tick the boxes, and report back at the end. Total time: about 30 minutes for installation + smoke tests, then a brief check-in the next morning to confirm the daily 08:00 sweep ran.

---

## Before you start (5-minute pre-flight)

Tick each box before moving to Step 1. If any are missing, stop and resolve them before installing — half the install problems come from missing prerequisites.

- [ ] You can log in to **Jira** at `myposgroup.atlassian.net` and you can read tickets in the `LEGAL` and `AIRD` projects.
- [ ] You can log in to **Outlook** with your myPOS account and you can send mail (i.e., your account isn't locked or in a "view only" state).
- [ ] You have **read + write** access to the [shared Legal SharePoint folder](https://mypos0.sharepoint.com/:f:/s/legal/IgD1ciPgZrRTQraRT4flBa4yAZC8NDvy0sv0YfcM6d3S_mw?e=WzgcuR). Open the link, try creating a new test folder, then delete it. If you can't create a folder, you don't have write access — ask the folder owner before continuing.
- [ ] You have a **Windows or Mac laptop** with internet access.
- [ ] You have **30 uninterrupted minutes**. The smoke test runs through real Jira tickets, so it's worth doing in one go.

---

## Step 1: Install Claude Code

1. Go to **[claude.ai/download](https://claude.ai/download)** and download the desktop app for your operating system.
2. Install it (standard installer — no special options).
3. Open Claude Code and sign in with your **myPOS** account (the same one you use for Jira and Outlook).

**Expected:** A chat-style window opens. The bottom of the window has a text input where you'll type slash commands.

**If something goes wrong:** Reach out to the AI Transformation team — issues at this stage are usually about laptop policy or VPN, not the Copilot itself.

- [ ] Claude Code installed and you're signed in.

---

## Step 2: Install the Legal Copilot plugin

In the Claude Code chat window, type each of these and press Enter (one at a time):

```
/plugin marketplace add myPOSTech/mypos-legal-marketplace
```

**Expected:** "Marketplace added: mypos-legal" (or similar). If you see an authentication prompt, follow it — your GitHub access is checked here. (If you don't have GitHub access set up, the AI Transformation team can fix this in a couple of minutes.)

```
/plugin install mypos-legal-copilot@mypos-legal
```

**Expected:** A confirmation prompt listing the plugin's permissions and components. Read through it once (so you know what's being installed), then confirm. After a few seconds you should see "Plugin installed: mypos-legal-copilot".

**If the install fails:** copy the error and skip to "Reporting back" at the end of this runbook — don't try to fix it yourself.

- [ ] Plugin installed successfully.

---

## Step 3: Connect Jira and Outlook

The first time you run a Copilot command, Claude Code will ask permission to connect to **Atlassian** (Jira) and **Microsoft 365** (Outlook + SharePoint). Trigger this now by typing:

```
/setup-copilot
```

You'll see one or two browser windows pop up:

1. **Atlassian login** — sign in with your myPOS account, then click "Allow" on the Atlassian permission screen.
2. **Microsoft 365 login** — same thing, sign in and approve the permissions.

You only do this once. After that, every Copilot command uses the same connection.

**If your laptop blocks the browser pop-ups** (some myPOS-managed laptops do), the Claude Code window will show the URL — copy it manually into your browser.

**If permissions look excessive:** they're not. The Copilot needs to read tickets, write comments, transition statuses, search emails, create drafts, and read/write the SharePoint folder. It cannot send emails — that's a hardcoded restriction inside the plugin.

- [ ] Atlassian connected.
- [ ] Microsoft 365 connected.

---

## Step 4: Finish setup

`/setup-copilot` is still running. After connecting both services, it will:

1. Ask you to **confirm the SharePoint folder**. The default points to the shared Legal folder. Press Enter to accept.
2. **Create the 10 matter-type subfolders** (NDAs, Contract Reviews, Regulatory Questions, etc.) inside the SharePoint folder, plus a `_knowledge/` subfolder for the team's shared brain. If any of these folders already exist, they're left untouched.
3. **Seed the knowledge files** in `_knowledge/`. Again, existing files are preserved — only missing ones are created.
4. **Register the daily run** for 08:00 Europe/Sofia, weekdays only.
5. **Run a smoke test** that confirms Jira, Outlook, SharePoint, and the Devil's advocate skill all respond.

**Expected final output:**

```
Setup complete.

✓ SharePoint root: https://mypos0.sharepoint.com/...
✓ 10 matter-type folders created (or preserved)
✓ Knowledge files seeded
✓ Scheduled run: weekdays 08:00 Europe/Sofia
✓ Atlassian connection: OK
✓ Microsoft 365 connection: OK
✓ Devil's advocate skill: available
```

If any line shows ✗ instead of ✓, **stop here** and skip to "Reporting back" — note exactly which line failed.

- [ ] All six lines show ✓.
- [ ] Open SharePoint in your browser and confirm the 10 matter-type folders + `_knowledge/` exist.

---

## Step 5: Smoke test #1 — triage a controlled test ticket

The point of this test is to put a known input through the Copilot and check the output looks right.

### 5a. Create a test ticket

In Jira, create a new ticket in the `LEGAL` project with this exact content (copy-paste):

> **Summary:** Pilot test — NDA review for Acme Test Corp
>
> **Description:**
> We've received a draft Mutual NDA from Acme Test Corp for a partnership exploration call next week. Please review.
>
> Counterparty: Acme Test Corp (Bulgarian entity)
> Purpose: Initial commercial discussions
> Term: 2 years
> Governing law: Bulgaria
>
> No tight deadline, but they're hoping to sign before Friday.
>
> *(This is a pilot test ticket — feel free to close after testing.)*

Make a note of the ticket key (e.g., `LEGAL-4567`).

### 5b. Run the Copilot on it

In Claude Code:

```
/triage LEGAL-4567
```

(use your actual ticket key)

### 5c. What to expect (in order, takes ~1–2 minutes)

The Copilot will narrate what it's doing. You should see, roughly in order:

1. "Loading shared knowledge from SharePoint..." → reads `_knowledge/*.md`
2. "Fetching ticket LEGAL-4567..."
3. "Deduplication check..." → likely "no duplicates found" since this is a fresh test ticket
4. "Classifying with `legal-triage-nda` skill..." → because the test ticket is an NDA
5. "Running Devil's advocate review..." → produces a verdict (probably `approve` or `revise`)
6. "Filing to SharePoint..." → uploads the draft + review to `NDAs/Pilot test - NDA review for Acme Test Corp/`
7. "Creating Outlook draft..."
8. "Posting Jira comment..."
9. "Updating ticket log..."

Final output: a summary block with the matter type, priority, risk flags (likely "none" for this test), and a confirmation that everything was filed.

### 5d. Verify the output

Open three browser tabs and check:

- [ ] **Jira ticket** — there's a new comment titled "AI Triage" with the matter type, priority, draft text, and a link to the SharePoint folder.
- [ ] **Outlook** — there's a draft in your "AI Drafts" folder (it may be auto-created if missing) titled "Re: Pilot test — NDA review for Acme Test Corp", containing the draft response with the `[DRAFT - FOR LAWYER REVIEW BEFORE SENDING]` banner at the top.
- [ ] **SharePoint** — open the `NDAs/` folder. Inside, there's a subfolder named `Pilot test - NDA review for Acme Test Corp`. Inside that, two files:
  - A `.docx` named like `LEGAL-4567_AcmeTestCorp_2026-04-27_v1.docx` (the draft)
  - A `.md` named like `LEGAL-4567_AcmeTestCorp_2026-04-27_v1_review.md` (the Devil's advocate review)

If any of these are missing or look wrong, note exactly what's missing.

- [ ] All three checks pass.

### 5e. Read the draft

Open the AI draft in Outlook and read it carefully. As a lawyer, ask yourself:

- Is the matter type correct? (Should be NDA.)
- Is the response addressed to the right party?
- Does it use hedging language ("our current view is…", "subject to confirmation")?
- Does it avoid making definitive legal commitments?
- Does it mention any individuals by name? (It shouldn't.)
- Does it reference Bulgarian law correctly? (This is a Bulgarian-jurisdiction NDA, so the Devil's advocate should have flagged any missing Commerce Act reference.)

Note any issues — these are valuable feedback for the team. Don't fix them in the draft yet; that's the next test.

---

## Step 6: Smoke test #2 — manual SharePoint filing

Add a fake "attachment" to the test Jira ticket: in Jira, drag any small file (a screenshot, a one-page PDF) onto the ticket as an attachment.

In Claude Code:

```
/file-to-sharepoint LEGAL-4567
```

**Expected:**

- The Copilot picks up the new attachment, files it to the existing SharePoint subfolder with the right naming convention (e.g., `LEGAL-4567_AcmeTestCorp_2026-04-27_attachment_<original-name>.<ext>`).
- A new comment on the Jira ticket lists the filed files with clickable SharePoint links.

Verify in SharePoint:

- [ ] The attachment is in the same subfolder as the draft and review, with the correct filename prefix.

---

## Step 7: Smoke test #3 — reply and close (with a test recipient!)

**This is the only command that actually sends an email**, so the recipient must be a test address.

Open the AI draft in Outlook. **Change the recipient to your own email address** (or another test address — never to a real client during pilot). Optionally edit a sentence or two so we can test the "lawyer edit detection" — for example, change "We are happy to review the NDA" to "We will review the NDA promptly." Save the draft (don't send it from Outlook).

In Claude Code:

```
/reply-and-close LEGAL-4567
```

**Expected:**

1. A confirmation prompt showing the recipient, subject, body preview, and a "material edits since AI draft: yes" line if you edited anything.
2. You type `y` to confirm.
3. The email is sent (you should receive it within a minute).
4. The sent `.eml` is filed to the same SharePoint subfolder as `..._final.eml`.
5. The Jira ticket transitions to **Done**.
6. The ticket log on SharePoint gets a new row marking the close.
7. If you made edits, a row is added to `feedback-log.md` capturing the diff between AI draft and what you sent.

Verify:

- [ ] You received the email at your test address.
- [ ] The Jira ticket is marked Done.
- [ ] The SharePoint subfolder has a new `..._final.eml` file alongside the draft and review.
- [ ] If you edited the draft: open `_knowledge/feedback-log.md` on SharePoint and confirm there's a new entry for `LEGAL-4567` describing your edit.

---

## Step 8: Wait for the morning sweep

The next weekday at **08:00 Europe/Sofia time**, the Copilot should run `/triage-board` automatically and process every To-Do ticket on the board.

The morning after the sweep, check:

- [ ] Open Claude Code — under "Recent activity" you should see a `legal-copilot-board-sweep` entry from earlier that morning, with a summary line ("Processed N tickets…").
- [ ] Open Jira — any tickets that were in To-Do status overnight should have new "AI Triage" comments.
- [ ] Open Outlook — your AI Drafts folder should have one draft per To-Do ticket from overnight.
- [ ] Open SharePoint — new ticket subfolders should appear in the appropriate matter-type folders.

If the morning sweep didn't run: in Claude Code, type `/schedule list` to see if the task is registered. If not, re-run `/setup-copilot` and try again.

---

## Reporting back

Write a short note (4–6 bullets) to the AI Transformation team covering:

1. **Did installation work end-to-end?** Yes / No / partial — describe.
2. **Smoke test results:** which of the three tests passed? For any failure, paste the error output and what step it happened at.
3. **Draft quality (subjective but important):**
   - Was the matter type correct?
   - Was the language appropriately hedged?
   - Did the Devil's advocate catch anything useful?
   - What would you have changed in the draft before sending?
4. **Filing structure:** did the SharePoint folders and filenames make sense to you? Anything you'd rename?
5. **The morning sweep:** did it run on time? Did the comments and drafts look ok?
6. **One thing that surprised you** (good or bad).

Send this to: **AI Transformation team** (Atanas Rusenov, Emil Gyorev). A short Teams message, an email, or a comment on the Jira ticket created for this pilot — all fine.

---

## Help, troubleshooting, and escalation

| If you see... | Do... |
|---|---|
| "Marketplace not found" on `/plugin marketplace add` | Check your GitHub access — you need read access to the `mypos-legal-marketplace` repo. |
| "Plugin install failed" with a generic error | Copy the full error, message AI Transformation. Don't retry — they need the original error message. |
| "Atlassian connection failed" during `/setup-copilot` | Sign out of Atlassian in your browser, then re-run `/setup-copilot`. The OAuth flow will restart. |
| "SharePoint folder not accessible" | Confirm you can open the [shared folder](https://mypos0.sharepoint.com/:f:/s/legal/IgD1ciPgZrRTQraRT4flBa4yAZC8NDvy0sv0YfcM6d3S_mw?e=WzgcuR) in your browser and create a test folder there. If you can't, ask the folder owner for write access. |
| "Devil's advocate skill not available" in the smoke test | This means the published reviewer skill isn't visible in your Claude Code install. Tell AI Transformation — they'll add the publisher marketplace or bundle a copy. |
| Outlook draft created but in the wrong folder | Note the actual folder name in your report. The Copilot creates "AI Drafts" if missing, but a localised mailbox might create it in a different language. |
| The morning sweep ran but processed 0 tickets | This is fine if the board was empty. Check `/schedule list` to confirm the next run is scheduled. |
| Anything else | Don't try to fix it. Note what happened and what you were trying to do. |

For anything urgent (security concern, accidentally sent email to a real client, ticket transitioned that shouldn't have): contact AI Transformation **immediately** in Teams.

---

## What to expect after the pilot

Assuming smoke tests pass and the pilot week goes well:

1. We'll address any feedback you raise (drafting style, folder names, missed risk flags).
2. We'll roll out to one or two more lawyers for a wider pilot.
3. After ~4 weeks, we'll publish v0.2 with whatever changes the pilot surfaced and onboard the rest of the legal team.

Thanks for being the first one through the door — your feedback over this pilot is what shapes the rollout.
