---
name: daily-triage
description: "Use this skill for daily email triage and matter activity review. Trigger on: 'daily triage', 'morning check', 'check my email', 'what's new today', 'inbox review', 'email triage', 'what came in', 'any new emails', 'catch me up', or any request to review recent emails and match them against open matters. Also trigger on scheduled runs. It produces a prioritized summary of new activity, flags urgent items, presents a structured decision list for unmatched emails, and auto-fills missing tracker fields discovered from email data."
---

# Daily Triage — Email & Matter Activity Review

## Purpose

Scan Gmail for recent emails, match them against open matters in the tracker, surface urgent items, and present a prioritized triage summary. It also detects missing tracker fields (like client email) from email data and auto-fills low-risk gaps. Its primary job is to tell you what happened and what needs attention.

## Conventions

- **"the lawyer"** refers to the user (you).
- **"Client"** refers to the person/entity who retained you on a matter.

## Dependencies

- **Matter tracker spreadsheet**: `matter-tracker.xlsx` — located using the same resolution logic as the other skills (CWD → parent → grandparent → ask).
- **Gmail MCP tools**: `gmail_search_messages`, `gmail_read_thread`, `gmail_read_message`. If Gmail tools are unavailable, skip the email scan and run only the tracker review (deadlines, stale matters).

## Workflow

### Step 1 — Load the Tracker

1. Find and load `matter-tracker.xlsx` (check CWD, parent, grandparent — ask if not found).
2. Read the **"Open Matters"** sheet. For each row, extract:
   - File # (A), Client Name (B), Matter Description (C), Last Activity (G), Opposing Party (H), Next Action (I), Client Email (M), Limitation Deadline (R), Court Deadlines (S), Other Parties (U)
3. Build a **name lookup index** — for each open matter, collect all searchable names:
   - Client name from column B (both the entity name and the individual name in brackets)
   - Opposing party from column H
   - Other parties from column U (split on commas)
   - Strip parenthetical role descriptions before matching (e.g., "Jane Rivera (co-plaintiff)" → match on "Jane Rivera")
4. Also read **"Closed Matters"** for the name index — recent closures may still have incoming email. But only include matters closed within the last 30 days.

### Step 2 — Scan Gmail

1. Search Gmail for emails from the **last 24 hours** (or since the last triage run if known):
   - Use `gmail_search_messages` with query: `newer_than:1d`
   - If this is a catch-up after a weekend or absence, the user may say "check since Friday" or "last 3 days" — adjust the time window accordingly.
2. For each email result, read the **full thread** using `gmail_read_thread` — snippets truncate critical details. However, if a thread has already been read during this triage session (same thread ID), skip it.
3. For each email/thread, extract:
   - **Sender** name and email address
   - **Subject** line
   - **Date/time** received
   - **Key content**: court dates, deadlines, settlement amounts, scheduling, instructions, or anything actionable
   - **Attachments**: note attachment filenames (don't download, just flag them)

### Step 3 — Match Emails to Matters

For each email, attempt to match it to an open matter:

1. **Match by email address**: Check if the sender's email matches any Client Email (column M) in the tracker.
2. **Match by name**: Check if the sender name or any name mentioned in the subject/body matches any name in the lookup index from Step 1.
3. **Match by keywords**: Check if the subject line contains matter-specific keywords from the Matter Description (column C) — e.g., a property address, a company name, a claim number.
4. **Match by reply chain**: If an email is a reply to a thread that matches a matter, the whole thread belongs to that matter.

Matching should be **case-insensitive** and support **partial matches** (last name matches are sufficient — "Smith" matches "John Smith" in column B).

Categorize each email as:
- **Matched** — clearly belongs to an open matter
- **Possibly matched** — partial or ambiguous match (e.g., common last name appears in multiple matters)
- **Unmatched** — no match to any open or recently-closed matter

### Step 4 — Classify Urgency

For each matched email, assign a priority:

**URGENT** (surface at top, with warning):
- Email from a court or tribunal domain (configure your jurisdiction's court email domains in CLAUDE.md — e.g. government or court system domains), or any email with "court" or "tribunal" in the domain
- Email referencing a hearing date, trial date, or court appearance
- Email containing the words "order", "endorsement", "judgment", or "ruling"
- Email with a deadline or due date within 7 days
- Email from opposing counsel with a settlement offer, demand, or time-limited proposal

**IMPORTANT** (surface prominently):
- Client email with instructions or questions requiring a response
- Email with an attachment that appears to be a legal document (claim, defence, affidavit, motion, contract)
- Email related to a matter whose limitation deadline is within 6 months

**ROUTINE** (list briefly):
- Administrative emails (scheduling confirmations, read receipts, FYI forwards)
- Newsletter, marketing, or system-generated emails
- Emails that are informational with no action required

### Step 5 — Fill Tracker Gaps from Email Data

For each matched email, compare what was found in the email/thread against what's in the tracker row. Detect and handle missing or outdated fields.

#### Auto-fill (no confirmation needed)

These fields are low-risk and objective. If the tracker field is blank and the email provides a clear value, write it directly to the tracker:

- **Client Email (column M)**: If blank and the matched email is clearly from the client (not opposing counsel, not a court), fill it with the sender's email address.
- **Opposing Party (column H)**: If blank and the email thread clearly identifies an opposing party by name (e.g., in a demand letter header, court filing, or "on behalf of [name]"), fill it.
- **Other Parties (column U)**: If blank and the email thread identifies additional parties (co-plaintiffs, co-defendants, insurers, agents), fill it. Append to existing values if the column already has content.

After auto-filling, note what was added in the triage output (see Step 8 format) so the user is aware.

#### Surface for review (confirmation needed)

These fields require judgment. Present them in the TRACKER GAPS section of the triage output for the user to approve:

- **Matter Description (column C)**: If the current description is vague or generic (e.g., just "Dispute" or "Legal matter") and the email thread reveals specifics (property address, claim type, transaction details), suggest an updated description.
- **Next Action (column I)**: If blank and the email thread implies an obvious next step (e.g., "please review and sign" → next action: "Review and sign documents"), suggest it.
- **Limitation Deadline (column R)**: If blank and the email thread references a limitation period or incident date from which one can be calculated, flag it with the suggested date and reasoning.

#### Implementation

Use openpyxl to write auto-fill values directly to the tracker file. Load with `load_workbook()` (not `data_only=True` — preserve formulas), update the cells, and save. Do not recalculate formulas — just save.

Keep a running list of all changes made for the triage summary. Format:
```
Auto-filled: 2026-012 | Smith — added client email (jsmith@example.com)
Auto-filled: 2026-031 | Acme Corp — added opposing party (Globex Inc.)
```

### Step 6 — Review Tracker for Alerts

Independent of the email scan, check the tracker for:

1. **Limitation alerts**: Any matter where column R (Limitation Deadline) is within 6 months. Calculate days remaining. Matters within 90 days get flagged as critical.
2. **Court deadline alerts**: Parse column S (Court Deadlines JSON) for any deadlines within 30 days.
3. **Stale matters**: Any matter where column G (Last Activity) is more than 21 days ago. These may need a follow-up or status check.
4. **Blank Next Action**: Any matter where column I (Next Action) is blank — these need a next step assigned.

### Step 7 — Classify Unmatched Emails

This is the most important step for unmatched emails. Instead of dumping them as a flat list, sort each unmatched email into one of four categories based on the thread content already read in Step 2.

**Deduplication check:** Before classifying an unmatched email, verify it truly has no match in the tracker. The matching in Step 3 uses client emails (column M), names, and keywords — but it can miss matches when:
- The client email column is blank (common for newer matters)
- The sender uses a different email than the one on file
- The matter was opened mid-session (after the tracker was loaded)

To catch these: before presenting the Inbox Review, re-scan the unmatched list one more time against the tracker using **broader matching** — check not just the sender name but also any names, company names, or addresses mentioned in the email body or subject against all tracker columns (B, C, H, U). If a match is found on this second pass, move the email to the appropriate matched section instead. Only truly unmatched emails should reach the Inbox Review.

**Category A — Active Matters (Not Tracked)**
Emails where you are clearly already acting as counsel — there is substantive legal correspondence, instructions given, work product exchanged — but the matter does not appear in the tracker. These are matters that should already be open but aren't.

Signals:
- You have replied with legal advice, a draft document, or strategic direction
- Thread contains retainer/engagement language or fee discussion
- Thread references a court file number, opposing party, or claim
- Multiple back-and-forth emails (not a one-off inquiry)

**Category B — New Client Inquiries (Retained or Near-Retained)**
Emails from someone who appears to be a new or prospective client discussing a legal issue for the first time. The intake is underway or recently completed, but no matter has been opened.

Signals:
- Sender describes a legal problem and asks for help
- You have responded with substantive guidance (not just "call me")
- Referral from another lawyer or referral platform with real case details
- Fee/retainer discussion has begun
- Client has sent supporting documents (contracts, court filings, screenshots)

**Category C — Leads (Not Yet Retained)**
Emails from potential clients who have made contact but where there is no indication of retention yet. These need follow-up but not a matter file.

Signals:
- First-contact email asking about services or describing a problem, but you haven't substantively responded yet
- Referral platform notification — lead only
- Website contact form submission
- Someone asking "do you handle [X]?" or requesting a consultation
- No fee discussion, no retainer, no documents exchanged

**Category D — Not Matters**
Everything else. Personal email, newsletters, marketing, billing/invoicing admin, spam, social media notifications, financial alerts, family correspondence.

Do not list Category D items individually — just note the count (e.g., "Plus 8 non-legal emails (newsletters, personal, admin) — skipped.").

### Step 8 — Present the Triage Summary

Output format:

```
Daily Triage — [date]

========================================
URGENT
========================================
[If any urgent items exist:]
- [COURT] 2026-012 | Smith — Email from Superior Court re: trial scheduling (received 9:41 AM) — OPEN AND READ
- [DEADLINE] 2026-031 | Acme Corp — 7-day cure period expires tomorrow (2026-03-21)

[If none:] Nothing urgent.

========================================
NEW ACTIVITY ON OPEN MATTERS
========================================
[Grouped by matter, showing matched emails:]

2026-012 | Smith — Damage deposit claim
  - 2 new emails:
    • From: opposing counsel (jane.doe@lawfirm.ca) — "Re: Settlement proposal" (10:15 AM)
    • From: court clerk — "Notice of Trial Date" (9:41 AM)
  - Suggested action: Read court email; respond to settlement proposal

2026-031 | Acme Corp — Corporate purchase
  - 1 new email:
    • From: client (contact@acmecorp.com) — "Signed docs attached" (2:30 PM) [2 attachments]
  - Suggested action: Review signed documents

[If no matters have new email:] No new email activity on open matters.

========================================
TRACKER UPDATES (auto-filled)
========================================
[If any fields were auto-filled in Step 5:]
- 2026-012 | Smith — Added client email: jsmith@example.com
- 2026-031 | Acme Corp — Added opposing party: Globex Inc.

[If none:] No gaps found.

========================================
TRACKER GAPS (needs your call)
========================================
[If any fields need your review from Step 5:]
- 2026-045 | Beta Ltd — Description is generic ("Dispute"). Suggest: "Commercial lease termination — 123 Main St, Toronto"
- 2026-008 | Lee — Limitation deadline appears to be 2026-09-15 (2-year from incident date 2024-09-15 per email thread). Add it? [y/n]

[If none:] No gaps requiring review.

========================================
TRACKER ALERTS
========================================
[Limitation deadlines:]
- 2026-012 | Smith — Limitation expires 2026-07-01 (103 days)
- 2026-044 | Patel — Limitation expires 2026-11-15 (240 days)

[Court deadlines within 30 days:]
- 2026-019 | Davis — Settlement conference 2026-04-02 (13 days)

[Stale matters (no activity > 21 days):]
- 2026-008 | Lee — Last activity 2026-02-15 (33 days ago)

[Blank next actions:]
- 2026-045 | Beta Ltd — No next action set

========================================
INBOX REVIEW
========================================

[Present Categories A, B, and C as a numbered decision list. Category D is summarized in one line at the end.]

ACTIVE MATTERS — NOT IN TRACKER
These are matters where you are already acting. They should be tracked.

  [1] Rivera / Delta Fitness — Globex dispute
      rivera@example.com | 3 emails today | You drafted a response to the demand letter
      → Ready to open: "new matter Delta Fitness"

  [2] Wayne Evans / MedCo — Physician NDA
      wevans@example.com | Confidentiality agreement finalized and sent today
      → Ready to open: "new matter Evans"

NEW CLIENTS — INTAKE UNDERWAY
Retention appears confirmed or near-confirmed based on correspondence.

  [3] Sidra Malik — Commercial lease dispute (Malik Corp)
      smalik@example.com | Client sent supporting docs, you reviewed and replied
      → Ready to open: "new matter Malik"

LEADS — NOT YET RETAINED
First contact only. Follow up or pass.

  [4] Referral platform lead — Condo building defect (construction/real estate, ON)
      Via referrals@example.com | Mold and water infiltration issue
      → Outside practice area? Skip or respond to decline.

Plus [N] non-legal emails (newsletters, personal, admin) — skipped.

---
Add to tracker? Reply with the numbers (e.g., "1, 2, 3") or "all" or "none".
To skip an item, just leave it out. You can always open it later with "new matter [name]".
```

### Step 9 — Handle the Decision

After the user responds to the decision list:

1. **For each selected number**: Confirm the client name that will be passed to the matter-tracker skill. The triage does NOT open the matter itself — it tells the user to run the command.
   - Example response: "Got it. Run these to open them:"
     - `new matter Delta Fitness` (Rivera — Globex dispute)
     - `new matter Evans` (Wayne — MedCo NDA)
     - `new matter Malik` (Sidra — lease dispute)

2. **For skipped items**: No action needed. They stay in Gmail and will surface again in the next triage if still unmatched.

3. **For leads (Category C)**: If the user selects a lead, note that there may not be enough email history yet for the matter-tracker's research procedure to build a full timeline. The matter-tracker will handle this gracefully — it will create a minimal entry and the user can update it as the intake progresses.

4. **If the user wants to act immediately**: They can run `new matter [name]` right after triage without waiting. The triage has already told them everything they need to know to decide — the matter-tracker skill will do its own full research pull independently.

## Important Rules

1. **Minimal writes only.** This skill auto-fills missing low-risk tracker fields (client email, opposing party, other parties) discovered from email data. It does not write timeline entries, briefs, or send emails. All auto-fills are reported in the triage output. Judgment calls (descriptions, deadlines, next actions) are surfaced for the user to approve, never written automatically.
2. **Read full threads, not snippets.** Snippets miss dates, amounts, and deadlines. Use `gmail_read_thread` on every matched thread.
3. **Don't skip unmatched emails.** Unmatched emails are valuable — they surface new client inquiries and matters that haven't been opened yet.
4. **Court emails are always urgent.** Any email from a court or tribunal domain gets top priority regardless of content.
5. **Be concise.** The triage should be scannable in under 60 seconds. One line per email, one line per alert. Details come later when the user asks to drill in.
6. **Handle weekends and gaps.** If the user runs this on Monday morning, expand the window to cover the weekend. If they say "I haven't checked since Thursday," search from Thursday.
7. **Don't duplicate the matter-tracker skill's job.** This skill identifies what needs attention and auto-fills missing contact/party fields. It does NOT write timeline entries, update Last Activity dates, or change matter status — those are the matter-tracker's job. The user runs "update matter [name]" for substantive tracker changes.
8. **Suggest but don't nag.** If stale matters or blank fields are found, mention them once. Don't repeat alerts the user has already seen in a previous triage.
9. **Gmail unavailable fallback.** If Gmail MCP tools aren't available, skip the email scan entirely and just run the tracker review (Step 6 alerts only). Tell the user: "Gmail tools not available — showing tracker alerts only." Steps 2-5 and 7 are skipped because they depend on email data.
10. **Classify, don't just list.** The Inbox Review section must categorize unmatched emails into A/B/C/D. Never present unmatched emails as a flat undifferentiated list. The categories exist to save the user decision-making energy — use the thread content to classify accurately.
11. **Category D stays quiet.** Non-legal emails (personal, newsletters, marketing, billing confirmations, spam) should never be listed individually. State the count and move on. The user does not need to see "LinkedIn — 3 new notifications" in their legal triage.
12. **Closed matter matches.** If an unmatched email matches a recently closed matter (within 30 days), flag it explicitly: "Matches closed matter 2026-043 (Miller, closed Mar 17)." This may indicate follow-up activity on a matter the user thought was done, or a returning client.
13. **Number the actionable items.** Categories A, B, and C must be presented as a single numbered list (not three separate numbered lists). This lets the user respond with "1, 3, 5" without ambiguity.
14. **One-line summaries with context.** Each numbered item gets: sender name + email, a one-line description of what the thread is about, and why it's in this category (e.g., "You drafted a response" or "First contact, no reply yet"). This gives the user enough to decide without re-reading the email.
15. **Auto-fill confidence threshold.** Only auto-fill a field when the match is unambiguous. If the matched email could be from the client OR a third party (e.g., a paralegal forwarding on behalf of a client), don't auto-fill — surface it in TRACKER GAPS instead. When in doubt, ask rather than write.
16. **Handle gap approvals inline.** If the user approves a suggested gap fill (e.g., responds "yes" to a limitation deadline suggestion), write it to the tracker immediately using the same openpyxl approach. No need to run `update matter` for a single field fix.
