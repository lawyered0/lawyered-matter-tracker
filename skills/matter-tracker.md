---
name: matter-tracker
description: "Use this skill whenever the user says 'new matter [name]', 'update matter [name]', or 'close matter [name]'. Also trigger on: 'show my open files', 'what's open', 'matter list', 'file list', 'CRM', 'matter tracker', 'conflict check', 'limitation period', 'court deadline', or any reference to tracking client files, opening/closing/updating matters, checking for conflicts, tracking limitation periods, or reviewing the status of legal work. Trigger even if the phrasing is casual, e.g. 'new matter Smith', 'update matter Jones', 'close matter Lee', 'run a conflict on Davis', 'what's the limitation on the Chen file'. Also trigger when the user uploads a .xlsx file and references client matters, or asks to pull emails for a client to update their file. Do NOT trigger on 'let's work on [name]', 'pull up [name]', or 'where are we with [name]' — those belong to the work-on-matter skill for loading context. Always use this skill in combination with the xlsx skill for spreadsheet operations."
---

# Matter Tracker — Open Files CRM

## Overview

This skill maintains a spreadsheet-based CRM of the user's open legal matters. It supports three core operations:

1. **Add** a new matter (with Gmail and client folder auto-population)
2. **Update** an existing matter (add notes, update description, log activity)
3. **Close** a matter (set status to Closed, record close date)

Plus a **Review** mode to display current open matters in conversation.

## Dependencies

- **xlsx skill**: Read the xlsx skill (`/mnt/skills/public/xlsx/SKILL.md` if available, or follow the xlsx skill's workflow) for spreadsheet creation/editing. **If the xlsx skill is unavailable**, use openpyxl directly following the formatting rules in this skill — the schema, row formatting, and column width specifications below are self-sufficient for direct openpyxl operations.
- **Gmail MCP tools**: Gmail tools are available as MCP tools (e.g., `gmail_search_messages`, `gmail_read_thread`, `gmail_read_message`). Use these directly — no loading step required. If Gmail tools are unavailable in the current environment, fall back to folder scan and manual entry.
- **Local file tools**: Use Glob, Grep, and Read tools to scan the client's matter folder for documents, correspondence, court filings, and other files that inform the timeline.

## Conventions

- **"the lawyer"** in timeline entries refers to the user. Always use "the lawyer" as shorthand for the user in timeline entries — e.g., "The lawyer sent demand letter to opposing counsel."
- **"Client"** refers to the person/entity who retained the lawyer on the matter.

## Spreadsheet Schema

The tracker spreadsheet uses two sheets: **"Open Matters"** (active files) and **"Closed Matters"** (archived files). Both sheets share the same column schema:

### Sheet 1: "Open Matters" — all active files
### Sheet 2: "Closed Matters" — archived files moved here on close

Both sheets use these columns:

| Column | Header | Format | Notes |
|--------|--------|--------|-------|
| A | File # | Text (e.g. "2026-001") | Auto-assigned: YYYY-NNN, incrementing from last entry |
| B | Client Name | Text | Primary client name. **Standard format: `Entity Name (Principal Name)`** — e.g. "Acme Corp Inc. (John Smith)". For individual clients with no entity, just use their name. For individuals acting through a numbered company, lead with the entity: "10014056 Holdings LLC (Jane D)". If multiple key individuals exist (e.g. two directors), comma-separate them inside the brackets: "ABC Real Estate Solutions Inc. (Bob Adams, Carol Chen)". **Never use slash format** (e.g. "Name / Corp") — always use the brackets format for consistency. This ensures the conflict check catches both the entity and the individual(s) behind it. |
| C | Matter Description | Text (wrap text) | Brief description of the engagement |
| D | Status | Text | "Open" or "Closed" |
| E | Date Opened | Date (YYYY-MM-DD) | Date the file was opened |
| F | Date Closed | Date (YYYY-MM-DD) | Blank until closed |
| G | Last Activity | Date (YYYY-MM-DD) | Updated on every interaction |
| H | Opposing Party | Text | If applicable; blank otherwise |
| I | Next Action / Deadline | Text (wrap text) | Key upcoming deadline or next step — see Next Action Format below |
| J | Timeline | Text (wrap text) | Concise chronological timeline built from Gmail and client folder files — see Timeline Format below |
| K | Client ID Verified | Text | "✓" once verified via identity verification service; "Pending" if not yet done |
| L | Conflict Check Done | Text | "✓" once conflicts check completed; "Pending" if not yet done |
| M | Client Email | Text | Client's email address; blank if not yet collected |
| N | Client Phone | Text | Client's phone number; blank if not yet collected |
| O | Client Address | Text | Client's mailing address; blank if not yet collected |
| P | Discovery Date | Date (YYYY-MM-DD) | Date of discovery for limitation purposes — ONLY set when there is a live or potential claim. Leave blank for transactional/advisory matters. |
| Q | Limitation Statute | Text | Key identifying the applicable limitation statute (e.g. "general_statute", "civil_rights"). ONLY set when there is a live or potential claim. Leave blank for transactional/advisory matters. |
| R | Limitation Deadline | Date (YYYY-MM-DD) | Calculated or manually entered limitation expiry. Auto-calculated from Discovery Date + Statute if both are set. ONLY set when there is a live or potential claim. |
| S | Court Deadlines | Text (JSON array) | Court-ordered deadlines stored as JSON. Each entry: {"date":"YYYY-MM-DD","description":"what's due","source":"endorsement or order reference"}. Only for bespoke deadlines from endorsements/orders — NOT routine rule-based deadlines like "defence due in 20 days". |
| T | Matter Folder | Text | Subfolder name (NOT a full path) within the Open Files directory (e.g. "Smith, J." — not "/Users/.../Smith, J."). Used to resolve the matter folder path on disk. When creating a new matter, search the Open Files directory for a subfolder matching the client — try ALL of: the individual's last name, first name, full name, "Last, First" format, the company/entity name, and common abbreviations. Client folders are often named after the entity rather than the person (e.g. "Acme Corp" not "Smith, John"). Cast a wide net: list all subfolders and grep for each search term separately. Write just the subfolder name. Leave blank if no match. |
| U | Other Parties / Related Persons | Text (wrap text) | All non-client, non-opposing parties involved in the matter: co-plaintiffs, co-defendants, witnesses, guarantors, landlords, agents, process servers, adjusters, corporate officers, and anyone else whose name should trigger a conflict check. Comma-separated. Include individuals behind corporate opposing parties if known (e.g. if opposing party is "Globex Corp", and the director is "Jane Doe", list "Jane Doe" here). This column is searched during conflict checks to catch indirect conflicts. |

### Formatting

- **Header row**: Bold, light blue fill (#D6E4F0), frozen pane, auto-filter enabled
- **Font**: Arial 10pt throughout
- **Column widths**: A=12, B=22, C=40, D=10, E=14, F=14, G=14, H=22, I=30, J=60, K=14, L=14, M=22, N=16, O=30, P=14, Q=30, R=14, S=40, T=50, U=50
- **Status column**: Use data validation (Open/Closed)
- **Limitation Statute column (Q)**: Use data validation — list = "general_statute,general_ultimate,consumer_protection,employment,civil_rights,construction,custom" *(Customize these statute keys and their limitation periods for your jurisdiction.)*
- **Wrap text** on columns C, I, J, S, and U **only** — do NOT set wrap_text on other columns

### Row Formatting (CRITICAL — must match existing rows exactly)

When appending or modifying any data row, **clone the formatting from the nearest existing data row** to ensure visual consistency. Specifically:

1. **Borders**: Every cell in columns A–U must have thin borders on all four sides (left, right, top, bottom). Use `Border(left=Side(style='thin'), right=Side(style='thin'), top=Side(style='thin'), bottom=Side(style='thin'))`.
2. **Wrap text**: Set `wrap_text=True` ONLY on columns C (Matter Description), I (Next Action / Deadline), J (Timeline), S (Court Deadlines), and U (Other Parties). All other columns must have `wrap_text=False` or no wrap setting.
3. **Font**: Arial 10pt, no bold (bold is header row only).
4. **Alignment**: Do not set vertical alignment to 'top' or any other value unless the existing rows use it. Match whatever the existing data rows use.

**Implementation pattern** (openpyxl):
```python
from openpyxl.styles import Font, Border, Side, Alignment

thin = Side(style='thin')
border = Border(left=thin, right=thin, top=thin, bottom=thin)
font = Font(name='Arial', size=10)
wrap_cols = {3, 9, 10, 19, 21}  # C, I, J, S, and U only

for col in range(1, 22):  # A through U
    cell = ws.cell(row=new_row, column=col)
    cell.font = font
    cell.border = border
    cell.alignment = Alignment(wrap_text=(col in wrap_cols))
```

**Why this matters**: If wrap_text or borders differ between rows, the tracker displays inconsistently in Excel/Sheets. Always inspect the last existing data row's formatting before writing a new one.

## Next Action Format

The Next Action / Deadline column (I) captures the single most important upcoming task or deadline on the file. Format:

```
YYYY-MM-DD: [brief description of next step or deadline]
```

If there is no specific date, omit the date prefix and just state the next step:

```
Draft claim and send to insurer as courtesy before filing
```

Rules:
- One entry only — the most critical next step
- Update on every interaction (new, update, close)
- On close, set to "FILE CLOSED" or leave blank
- Prioritize court deadlines, limitation periods, and filing deadlines over internal to-dos

Examples:
```
2026-03-27: Settlement conference at 1:15 PM
2026-03-19: 7-day cure period expires; file for judgment
Send draft claim to insurer with response deadline
Awaiting client instructions re: citizenship oath
```

## Timeline Format

The Timeline column (J) has two parts: a **substantive summary** at the top, then the **chronological log** below it. Both are mandatory.

### Part 1 — Substantive Summary

Every timeline begins with a `SUMMARY:` block that captures the substance of the matter — the facts, the money, the legal issues, and the current posture. This is the "what is this about and where does it stand" overview that someone reading the tracker needs to understand the file without opening a single document.

```
SUMMARY: [Client] retained [Your Name] re: [what happened — the underlying facts, not just "breach of contract" but the actual story]. [Key amounts, terms, or stakes]. [Current posture — what stage the matter is at, what's been done, what's pending]. [Key risk or issue if any].
```

Rules for the summary:
- **3-6 sentences**. Enough to understand the matter; short enough to scan.
- **Lead with the facts**, not the legal category. "Client paid $5,000 deposit for rental unit; landlord kept deposit and went silent" is useful. "Breach of contract / deposit dispute" is not — that's what the Matter Description column is for.
- **Include the money**. Claim amounts, deal values, settlement ranges, invoices — whatever is at stake.
- **Include key parties and their roles** beyond just client/opposing. Who's the landlord, the broker, the witness, the insurer?
- **State the current posture**. "Demand letter sent; 10-day deadline expires March 27" or "SPA executed; post-closing items outstanding" or "Defence filed; awaiting trial date."
- **Flag the key risk or open issue** if there is one. "Limitation expires July 1" or "Client's evidence is mostly oral — no written contract" or "Seller's counsel pushing back on earnout terms."
- **No legalese**. Write it like you're explaining the file to a smart non-lawyer.
- **Update the summary on every update/close** to reflect current state. The summary should always be current — it's a snapshot, not a historical record.

### Part 2 — Chronological Log

Below the summary, the dated entries form the procedural timeline:

```
YYYY-MM-DD: [one-line summary of what happened]
YYYY-MM-DD: [next event]
YYYY-MM-DD: [next event]
```

Rules for the chronological log:
- One line per significant event (email sent, document received, call referenced, deadline set, etc.)
- Chronological order, oldest first
- Each line is date + colon + short plain-language summary (no legalese, no fluff)
- Keep each entry to ~10-15 words max
- Include who did what where relevant (e.g. "Client sent signed SPA to opposing counsel")
- Use "the lawyer" to refer to the user — e.g. "The lawyer sent demand letter"
- **Always include the intake/retention event** as the first timeline entry (e.g. "Client retained the lawyer re: ...")
- **Always include filing events** (e.g. "The lawyer sent claim to process server for filing and service")
- If closing, append: `YYYY-MM-DD: FILE CLOSED.`
- On update, merge new entries into the existing timeline in chronological order — never duplicate or delete existing entries
- **Long timelines**: Never compress or summarize timeline entries — keep every entry at full granularity regardless of length. If the cell gets unwieldy in Excel, that's acceptable; the complete record is more valuable than a tidy spreadsheet.

### Full Example

```
SUMMARY: John Smith paid a $3,000 deposit and lent $5,000 cash to Jane Doe and Bob Wilson for rental arrangements. Doe repaid $2,500; balance of $6,311.50 remains unpaid including $300 e-Transfer and car rental expenses. Wilson acknowledged the debt in a recorded conversation (Sept 2024) but defendants went silent afterward. Demand letter drafted and approved; on hold pending client's schedule. Limitation expires July 1, 2026.

2026-02-25: Client retained the lawyer; intake and matter summary provided
2026-03-01: Client provided contracts, transfer proofs, bank statements, recording transcript
2026-03-02: The lawyer reviewed evidence; confirmed sufficient for demand
2026-03-05: Client confirmed claim amount $6,311.50; authorized demand letter
2026-03-05: Invoice issued ($571.73); payment received
2026-03-12: The lawyer sent final demand draft; client approved but on hold
```

### Backfilling Summaries on Update

When running **update matter** and the existing timeline does **not** start with `SUMMARY:`, generate a summary from all available sources:

1. Read the existing timeline entries for procedural context
2. Read the Matter Description (column C) for scope
3. Read the `_matter-brief.md` if it exists
4. Use the Gmail threads and folder documents already pulled during the research procedure
5. Draft a summary following the rules above and prepend it to the timeline

Present the proposed summary to the user for confirmation along with the rest of the update. The user may want to add facts or correct emphasis.

## Audit Columns (K–L)

Columns K and L are simple compliance checkmarks set on every new matter:

- **K — Client ID Verified**: Set to "✓" on file opening. ID verification is performed via the firm's identity verification service. Default to "✓" for all new matters — the user verifies ID as standard intake practice.
- **L — Conflict Check Done**: Set to "✓" on file opening. Confirms a conflicts check was completed before the file was opened. Default to "✓" for all new matters.

These columns do not require Gmail searches. They are set automatically when a new matter is created and carried over when a matter is closed.

## Client Contact Columns (M–O)

Columns M, N, and O store the client's email, phone, and address. **Actively extract these during the Gmail pull and folder scan** — treat contact info as a first-class extraction target on every research pull, not an afterthought:

- **Email (M)**: Extract the client's email address from the "From" header of their first email, or from engagement letters / intake forms in the folder. If multiple email addresses are found, use the one the client communicates from most.
- **Phone (N)**: Look for phone numbers in email signatures, engagement letters, intake forms, and the body of early correspondence. Scan email bodies and signatures for patterns like `(XXX) XXX-XXXX`, `XXX-XXX-XXXX`, `+1-XXX-XXX-XXXX`. Also check court filings (claims list party addresses and sometimes phone numbers) and any PDF intake forms in the folder. Include area code.
- **Address (O)**: Look for mailing addresses in engagement letters, intake forms, court filings (e.g., claims list the plaintiff's address), email signatures, and statement of claim cover pages. Also check any correspondence that includes a letterhead or return address from the client.

**Always read engagement letters and intake forms** even if they don't seem timeline-relevant — these are the richest source of contact info. If columns M-O are blank after the research pull, explicitly note this in the confirmation output so the user can provide the info manually.

These columns should be populated for every matter where the information is findable.

## Limitation Period Columns (P–R)

Columns P, Q, and R track limitation periods. **These are ONLY populated when there is a live or potential claim.** Do NOT set these for transactional, advisory, or corporate matters where no cause of action exists.

- **P — Discovery Date**: The date of discovery that triggers the limitation clock. This is a judgment call — flag if ambiguous and ask the user.
- **Q — Limitation Statute**: One of: `general_statute` (2yr), `general_ultimate` (10yr), `consumer_protection` (2yr), `employment` (2yr), `civil_rights` (1yr), `construction` (2yr), or `custom`. **Customize these statutes and their periods for your jurisdiction.** If none of the named statutes apply, use `custom` and ask the user to provide the deadline manually.
- **R — Limitation Deadline**: Auto-calculated from P + Q. If the statute is `custom`, the user enters the deadline manually.

When creating a new matter that involves a claim (e.g. Small Claims, demand letter, employment dispute), **always ask the user about the discovery date and applicable limitation** if not obvious from the emails. Flag any limitation period that is within 6 months of expiry.

## Court Deadlines Column (S)

Column S stores court-ordered deadlines as a JSON array. Each entry has three fields:

```json
[
  {"date": "2026-03-25", "description": "Amend claim to add corporation", "source": "March 12 endorsement"},
  {"date": "2026-04-27", "description": "Serve Form 1B on defendants", "source": "Rule 1.03 — 30 days before trial"}
]
```

**Only enter bespoke deadlines** from endorsements, orders, or case-specific requirements — NOT routine rule-based deadlines that the lawyer already knows (e.g. "defence due in 20 days", "disclosure 14 days before settlement conference"). The purpose of this column is to capture the one-off deadlines that come out of specific judicial endorsements and could be missed.

When updating a matter, if a Gmail search or folder scan reveals a new court date, endorsement, or order with a deadline, add it to the JSON array and alert the user.

## Other Parties / Related Persons Column (U)

Column U captures every person or entity involved in the matter who is NOT the client (column B) or the primary opposing party (column H). This includes:

- Co-plaintiffs and co-defendants
- Witnesses (including expert witnesses)
- Guarantors, indemnifiers, sureties
- Landlords, agents, brokers
- Corporate officers / directors behind opposing entities
- Lawyers and paralegals for opposing parties
- Process servers, adjusters, mediators
- Family members or related individuals (e.g., "brought on behalf of" parties)

**Format**: Comma-separated names. Include role context where helpful: "Jane Rivera (co-plaintiff/witness), David Park (opposing counsel, Park Law)".

**Why this matters**: The conflict check searches this column. If a future prospective client's name appears here, the user is alerted before opening a file that could create a conflict.

## Core Research Procedure

**Every command (new, update, close) begins with a Gmail pull, then a client folder scan.** These are the universal first steps:

### Step A — Gmail Search

Gmail provides the primary chronological backbone of the timeline — it captures communications, instructions, scheduling, and references to key events.

1. Use the Gmail MCP tools directly (e.g., `gmail_search_messages` with a query for the client name).
2. Search for emails matching the client name. Cast a wide net:
   - Search the client's name
   - If known, also search the opposing party name or matter keywords
   - For **new matter**: use **no time limit** by default. Paginate through results to find the earliest correspondence. If results exceed 3 pages, ask the user: "I'm finding emails going back to [date]. Should I keep going deeper or is that far enough?"
   - For **update matter**: search from the Last Activity date onward
   - For **close matter**: search from the Last Activity date onward
3. Read email threads until the timeline is complete. **Use `gmail_read_thread` or `gmail_read_message` on each thread** — do not rely on snippets alone, as snippets truncate critical details like dates, times, locations, and court file numbers. Read every thread that could contain a timeline event. There is no cap on how many threads to read — the goal is a complete chronological record. If there are dozens of threads, read them all. For efficiency, prioritize court/scheduling emails first, then substantive correspondence, then administrative emails — but do not skip threads just because there are many.
4. Extract: dates, key actions, parties involved, documents exchanged, deadlines, outcomes.
5. **Extract client contact info**: On every Gmail pull, look for the client's email address (from the "From" header), phone number, and mailing address (from email signatures, body text, or attached documents). If found and columns M-O are blank, populate them.
6. **Court and scheduling emails are highest priority.** When any email originates from a court or tribunal domain (configure your jurisdiction's court email domains in CLAUDE.md — e.g. government or court system domains, or any court clerk address) or references a court file number, scheduling, or hearing date, **always read that message in full** and extract all dates, times, locations, and Zoom/video links. These must be captured verbatim in the Next Action field (with exact date and time) and in the Timeline. Never summarize or skip a court scheduling email.
7. Build the base timeline from Gmail results.

### Step B — Client Folder Scan

The client's matter folder must be located and scanned for document-level evidence. Use the Matter Folder name from column T (or, for a new matter, search the Open Files directory for a subfolder matching the client name) to find the correct folder. Then scan that folder and its immediate subdirectories.

**Scope by operation:**
- **New matter**: Scan all files in the folder (full history needed).
- **Update matter**: Focus on files modified since the Last Activity date from the tracker. Still list all files via Glob, but only read/process those with modification dates after Last Activity.
- **Close matter**: Same as update — only files modified since Last Activity.

**Multi-matter folders:** Client folders often contain subfolders for separate matters (e.g. "Real Estate Purchase/", "Small Claims - Damage Deposit/"). If the current working directory contains matter-specific subfolders, identify which subfolder corresponds to the matter being tracked (match by matter description or keywords) and scope the scan to that subfolder. If the user is already inside the correct subfolder, scan from there. If ambiguous, ask the user which subfolder to use.

**Steps:**

1. Use **Glob** to list files in the current working directory and subdirectories. Look for common legal file types: `**/*.pdf`, `**/*.docx`, `**/*.doc`, `**/*.xlsx`, `**/*.txt`, `**/*.msg`, `**/*.eml`.
2. For **update/close**: filter to files modified since the Last Activity date. For **new matter**: consider all files.
3. Use file names, creation dates, and modification dates to infer timeline events. File names in a law practice are often descriptive (e.g. "Statement of Claim - Filed 2026-01-15.pdf", "Engagement Letter - Smith.docx", "Settlement Conference Brief.pdf").
4. Where helpful, **Read** key documents (PDFs, Word docs) to extract:
   - Dates of filings, service, correspondence
   - Party names and roles
   - Court dates, endorsements, deadlines
   - Client contact information (from engagement letters, intake forms)
   - **Names of other parties** for column U (witnesses, co-parties, agents, corporate officers)
5. Read every document that could contain a timeline event or contact info. Prioritize by relevance:
   - Engagement/retainer letters (intake date, scope, client contact info — **always read these**)
   - Filed court documents (claims, defences, motions — filing dates and deadlines)
   - Endorsements and orders (court-ordered deadlines)
   - Correspondence (demand letters, settlement offers — key milestones)
   - Intake forms and client-provided documents (contact info, background facts)
   - Skip only clear duplicates (e.g. "Draft v1", "Draft v2", "Draft v3" — read only the final) and purely administrative files (invoices, receipts) unless file names suggest they contain date/event info
6. Look specifically for events the Gmail timeline missed — folder files often capture things like filed documents, executed agreements, and court endorsements received in person or by mail.

### Step C — Merge Sources

1. Start with the Gmail-based timeline as the backbone
2. Merge in any additional events found in folder files, in chronological order
3. De-duplicate: if a folder file and an email describe the same event, keep one entry (prefer the more precise date)
4. Gmail captures most events; folder files fill gaps (e.g. documents received by mail, filed originals, endorsements picked up at court)

If Gmail tools are unavailable, build the timeline from folder contents alone and inform the user. If the folder is empty or contains no relevant files, rely on Gmail alone. If both are unavailable, ask the user to provide details manually.

## Duplicate / Conflict Check

Before adding a new matter, **always run a full conflicts check** against the existing tracker. This has two parts: a duplicate check (same client) and an adverse interest check (cross-party conflicts).

### Part 1 — Duplicate Check (same client name)

1. Load the tracker (see "Finding the Tracker" below).
2. Search the "Open Matters" sheet for the client name (case-insensitive partial match on column B — Client Name).
3. Also check "Closed Matters" if the sheet exists.
4. If a match is found:
   - If on Open Matters: **stop and ask** — "There is already an open file for [Name] (File #[X]). Did you mean to update that file, or is this a separate matter?"
   - If on Closed Matters only: **flag but proceed** — "Note: [Name] had a previously closed file (File #[X]). Opening a new file."

### Part 2 — Adverse Interest Check (cross-party conflicts)

5. If the new matter has an opposing party, search **all rows on both sheets** for that opposing party name in columns B (Client Name), C (Matter Description), and U (Other Parties). This catches the case where someone you're suing (or negotiating against) is an existing or former client, or was involved in another matter.
6. Also search columns H (Opposing Party) and U (Other Parties) across all rows for the **new client's name**. This catches the case where the new client was previously on the other side of one of your matters.
7. Also search column C (Matter Description) for the new client's name — descriptions sometimes mention parties not captured elsewhere.
8. If any adverse match is found: **stop immediately and alert the user** — "Potential conflict: [New Opposing Party] appears as a client in File #[X], or [New Client] appears as an opposing party in File #[X]. You must resolve this conflict before opening this file."
9. If no matches on either part, proceed normally.

**Search scope summary** — the conflict check searches these columns for every name involved:
- Column B (Client Name) — both sheets
- Column C (Matter Description) — both sheets
- Column H (Opposing Party) — both sheets
- Column U (Other Parties / Related Persons) — both sheets

## Workflows

### 1. NEW MATTER

**Trigger**: "new matter [name]" or "new matter"

**Steps**:

1. Extract the client name from the command. If absent, ask.
2. **Run the Duplicate / Conflict Check** against the existing tracker.
3. **Run the Core Research Procedure** (folder scan + Gmail, no time limit — paginate to find full history). **Both steps (Gmail search AND folder scan) must be completed before drafting the timeline.** Do not skip the folder scan — even if Gmail provides a rich history, the folder often contains court documents, endorsements, and filed originals that Gmail misses. The folder scan also confirms the Matter Folder name for column T.
4. From the folder files and emails, draft:
   - Client Name — **use the standard format: `Entity Name (Principal Name)`**. If the client is a corporation or other entity, identify the principal/directing mind from the correspondence or folder files. If you can't identify the principal from the documents, ask the user: "Who is the principal/directing mind of [entity]?" For individual clients with no entity, just use their name.
   - Matter Description (one-line summary of the engagement)
   - Opposing Party (if identifiable)
   - Next Action / Deadline (the most critical upcoming step)
   - Timeline — **both parts**: substantive summary (SUMMARY: block) AND chronological log from folder files and emails (include retention/intake and filing events)
   - Other Parties (anyone else involved — co-parties, witnesses, lawyers, agents)
   - Client Email / Phone / Address (if found in emails or folder files)
5. **Present to user for confirmation:**
   ```
   New matter for [Name]:
   Matter: [description]
   Opposing: [party or "N/A"]
   Other Parties: [list or "None identified"]
   Next Action: [deadline or next step]
   Contact: [email] | [phone] | [address] (or "not found" for each)

   Timeline:
   SUMMARY: [substantive summary — facts, money, posture, risk]

   YYYY-MM-DD: [event]
   YYYY-MM-DD: [event]
   ...

   Add to tracker? Any corrections?
   ```
6. After confirmation:
   - Load the tracker (see "Finding the Tracker"), assign next File #, append row
   - If no tracker exists -> create new tracker from template in the Open Files directory, then add row
   - Set Status = "Open", Date Opened = earliest timeline date (or today), Last Activity = today
   - Set Client ID Verified = "✓", Conflict Check Done = "✓"
   - Populate Client Email/Phone/Address (columns M-O) if available from folder files or emails
   - Populate Other Parties (column U) with all non-client, non-opposing parties identified
   - **If the matter involves a claim**: ask about discovery date and limitation statute; populate columns P-R. Flag if limitation is within 6 months.
   - **If the matter is transactional/advisory**: leave columns P-R blank.
   - Leave Court Deadlines (S) blank unless folder files or emails reveal a specific court-ordered deadline.
   - **Matter Folder (T)**: Search the workspace directory for a subfolder matching the client. **All matching must be case-insensitive.** First, dump the full directory listing to a text file using `ls -1 > /tmp/dirlist.txt`, then grep against that file — this avoids shell issues with special characters (colons, ampersands, parentheses, etc.) in folder names. Try matching against ALL of these permutations of the client name: "First Last" (e.g. "wayne evans"), "Last, First" (e.g. "Evans, Wayne"), "Last First" (no comma), just the last name, just the first name, and any company/entity name from the matter description or opposing party field. **Also search for the mother/father/third-party name if the matter is brought on someone else's behalf** (e.g. for "Smith v. Jones" brought by Mary Davis on behalf of Ann Smith, search for "Davis", "Mary", "Smith", and "Ann"). Folders are often named in lowercase or informal formats (e.g. "wayne evans" not "Evans, Wayne"), or after the entity rather than the person (e.g. "Acme Corp" not "Smith, John"), and frequently contain special characters like colons (e.g. "Mary Davis : Smith"). Cast a wide net — grep each search term separately and case-insensitively against the text file listing. If found, write just the subfolder name exactly as it appears on disk. If not found, leave blank.
7. Save the updated tracker to disk.

### 2. UPDATE MATTER

**Trigger**: "update matter [name]" or "update matter"

**Steps**:

1. Extract the client name. If absent, ask.
2. Load the tracker (see "Finding the Tracker"). Find matching row (case-insensitive partial match; if ambiguous, ask).
3. **Run the Core Research Procedure** (folder scan + Gmail from Last Activity date onward).
4. Read the existing Timeline from the spreadsheet.
5. **Check for substantive summary.** If the existing timeline does not start with `SUMMARY:`, generate one now — use the existing timeline entries, the Matter Description (column C), the `_matter-brief.md` if it exists, and the Gmail threads and folder documents from step 3. If a summary already exists, update it to reflect the current state of the matter (the summary should always be current, not historical).
6. Merge new events into the existing chronological log in chronological order. Do not duplicate or remove existing entries.
7. **Present the updated timeline to user for confirmation:**
   ```
   Updated timeline for [Name]:

   [If summary was added/updated:]
   SUMMARY: [substantive summary — updated to current state]

   [existing dated entries]
   [NEW] YYYY-MM-DD: [new event]
   [NEW] YYYY-MM-DD: [new event]

   Next Action: [updated next step/deadline]
   Also update matter description? Currently: "[current]"
   Confirm?
   ```
8. After confirmation:
   - Write the full timeline (summary + chronological log) to the Timeline column
   - Update Last Activity to today
   - Update Next Action / Deadline
   - Update Matter Description if scope changed
   - Update Client Email/Phone/Address if new contact info found in folder files or emails
   - Update Other Parties (column U) if new parties were identified
   - If folder files or emails reveal a new court date, endorsement, or order with a deadline, add it to the Court Deadlines JSON (column S) and alert the user
   - **Prune expired court deadlines**: When updating column S, remove any entries whose date has passed and note to the user which deadlines were cleared (e.g., "Removed expired deadline: 2026-02-15 — Amend claim"). This keeps the column focused on upcoming obligations.
   - If the matter now involves a claim but limitation columns (P-R) are blank, flag this and ask the user about discovery date and statute
   - **If column T (Matter Folder) is blank**, attempt to populate it now using the folder resolution logic from the NEW MATTER workflow.
9. Save the updated tracker to disk.
10. **Write/refresh `_matter-brief.md`** in the matter folder (column T). If the brief exists, read it and update with current information. If it doesn't exist, create it. The brief is a short (~1 page max) current-state snapshot of the matter — see the format below under "Matter Brief Format." This step happens automatically after saving the tracker — no need to ask the user for separate confirmation.

### 3. CLOSE MATTER

**Trigger**: "close matter [name]" or "close matter"

**Steps**:

1. Extract the client name. If absent, ask.
2. Load the tracker (see "Finding the Tracker"). Find matching row on the "Open Matters" sheet.
3. **Run the Core Research Procedure** (folder scan + Gmail from Last Activity date onward) — capture any final correspondence.
4. Merge any new events into the existing timeline.
5. Append `YYYY-MM-DD: FILE CLOSED.` as the final timeline entry.
6. **Populate any blank columns before closing:**
   - If column T (Matter Folder) is blank, attempt to populate it using the folder resolution logic (the folder scan in step 3 already identified the path — write the subfolder name).
   - If columns M-O are blank but contact info was found during the research pull, populate them now.
   - If column U is blank but other parties were identified, populate it now.
7. **Present to user for confirmation:**
   ```
   Closing file for [Name] — [Matter Description]
   Final timeline:
   [full merged timeline including CLOSED entry]

   This will move the matter to the "Closed Matters" tab.
   Confirm close?
   ```
8. After confirmation:
   - Update the row: final timeline, Status = "Closed", Date Closed = today, Last Activity = today, Next Action = "FILE CLOSED"
   - **Move the row to the "Closed Matters" sheet:**
     1. If the "Closed Matters" sheet does not exist, create it with the same header row formatting as "Open Matters" (bold, light blue fill, frozen pane, auto-filter, same column widths)
     2. Copy all cell *values* from the row on "Open Matters" to the next available row on "Closed Matters"
     3. Apply the standard row formatting (borders, font, wrap text on C, I, J, S, and U only — see Row Formatting section)
     4. Delete the row from "Open Matters"
   - **Important**: When deleting from "Open Matters", do NOT delete the header row. If the closed matter is the only data row, the sheet should have just the header row remaining.
9. Save the updated tracker to disk.
10. **Update `_matter-brief.md`** in the matter folder — append "FILE CLOSED" to the summary and mark open items as resolved or moot. If no brief exists, create a final one for the closed file.

### 4. REVIEW OPEN MATTERS

**Trigger**: "show my open files", "what's open", "matter list", "file summary"

**Steps**:

1. Load tracker.
2. Read the "Open Matters" sheet.
3. Display a clean summary in conversation:

```
You have X open matters:

1. 2026-001 | Smith, J. | Share purchase agreement | Next: Awaiting signed docs | Last: 2026-03-01
2. 2026-002 | Chen, R. | Damage deposit — Small Claims | Next: 2026-04-15 Settlement conf. | Last: 2026-03-10
...
```

4. Flag any upcoming deadlines within the next 30 days (court deadlines from column S and Next Action dates from column I).
5. **Flag any limitation periods within 6 months** (column R). Limitation deadlines are the highest-priority alerts — display them prominently, e.g.: "LIMITATION: File #2026-002 (Chen) — limitation expires 2026-06-15 (89 days)."
6. Ask if the user wants to update or close any of them.

**Filter support**: If the user asks a targeted question — e.g. "which matters have limitation deadlines in the next 90 days", "what hasn't been touched in 30 days", "show me all Small Claims files", "matters with upcoming court dates" — filter the display accordingly:

- **By staleness**: Filter by Last Activity (column G) — e.g. matters not touched in X days.
- **By limitation urgency**: Filter by Limitation Deadline (column R) — e.g. deadlines within X months.
- **By court deadlines**: Filter by Court Deadlines (column S) — e.g. hearings within X days.
- **By matter type**: Grep Matter Description (column C) for keywords — e.g. "Small Claims", "lease", "employment".
- **By party**: Search columns B, H, and U for a name.
- **By status**: Open vs. Closed (cross-sheet).

### 5. REVIEW CLOSED MATTERS

**Trigger**: "show closed files", "closed matters", "what have we closed", "archived matters"

**Steps**:

1. Load tracker.
2. Read the "Closed Matters" sheet. If it doesn't exist or is empty, inform the user.
3. Display a clean summary in conversation:

```
You have X closed matters:

1. 2025-003 | Lee, D. | Lease dispute — Small Claims | Opened: 2025-09-01 | Closed: 2026-01-15
...
```

### 6. CONFLICT CHECK (standalone)

**Trigger**: "conflict check [name]", "run a conflict on [name]", "conflicts check"

**Steps**:

1. Extract the name to check from the command. If absent, ask.
2. Load the tracker (see "Finding the Tracker").
3. Run both parts of the Duplicate / Conflict Check (see above), treating the provided name as both a potential client name AND a potential opposing party name:
   - Search column B (Client Name) on both sheets for the name.
   - Search column C (Matter Description) on both sheets for the name.
   - Search column H (Opposing Party) on both sheets for the name.
   - Search column U (Other Parties) on both sheets for the name.
4. Report results clearly:

```
Conflict check for "[Name]":

[If matches found:]
  - File #2026-001: [Name] is the CLIENT (matter: [description], status: [open/closed])
  - File #2026-003: [Name] is the OPPOSING PARTY (client: [client name], matter: [description], status: [open/closed])
  - File #2026-005: [Name] appears in OTHER PARTIES (client: [client name], matter: [description], role: [if known])

[If no matches:]
  No conflicts found. "[Name]" does not appear as a client, opposing party, or related person in any open or closed matter.
```

5. This workflow is read-only — it does not modify the tracker.

### 7. REOPEN MATTER

**Trigger**: "reopen matter [name]", "reopen [name]", "reactivate matter [name]"

**Steps**:

1. Extract the client name. If absent, ask.
2. Load the tracker (see "Finding the Tracker"). Find matching row on the **"Closed Matters"** sheet. If the matter is already on Open Matters, inform the user it's already open.
3. **Present to user for confirmation:**
   ```
   Reopening: File #[X] | [Client Name] — [Matter Description]
   Closed on: [Date Closed]

   This will move the matter back to "Open Matters" and set Status to Open.
   Confirm reopen?
   ```
4. After confirmation:
   - Copy all cell values from the row on "Closed Matters" to the next available row on "Open Matters"
   - Apply the standard row formatting (borders, font, wrap text — see Row Formatting section)
   - Update: Status = "Open", Date Closed = blank, Last Activity = today
   - Update Next Action / Deadline — ask the user what the next step is, since the previous "FILE CLOSED" entry is no longer valid
   - Append to the timeline: `YYYY-MM-DD: FILE REOPENED.`
   - Delete the row from "Closed Matters"
5. Save the updated tracker to disk.
6. Suggest running "update matter [name]" to do a fresh Gmail + folder scan and rebuild the timeline.

## Template Creation

When creating a new tracker from scratch, use openpyxl to build the spreadsheet with:

**Sheet 1 — "Open Matters":**
- Header row with formatting per the schema above (columns A-U)
- Freeze panes at row 2
- Auto-filter on the header row
- Data validation on column D (Status): list = "Open,Closed"
- Data validation on column Q (Limitation Statute): list = "general_statute,general_ultimate,consumer_protection,employment,civil_rights,construction,custom"
- Print area and page setup: landscape, fit to 1 page wide

**Sheet 2 — "Closed Matters":**
- Same header row formatting as "Open Matters" (bold, light blue fill #D6E4F0, same column widths, frozen pane, auto-filter)
- Same column schema (A-U)
- Tab color: grey (#808080) to visually distinguish from active sheet
- Initially empty (header row only)

Use the xlsx skill's recalc script if any formulas are added.

## File Number Assignment

- Format: `YYYY-NNN` where YYYY is the current year and NNN is a zero-padded sequential number
- If the tracker is new, start at `{current_year}-001`
- If appending, scan **all** File # values across both Open and Closed Matters sheets to find the highest number for the current year, then increment by 1. This prevents collisions when files have been closed and removed from Open Matters.
- If the year has changed since the last entry, reset to `{new_year}-001`

## Important Behaviour Rules

1. **Every command does a full research pull (Gmail + folder scan).** New, update, and close all search Gmail and scan the client folder to build/refresh the timeline. No exceptions.
2. **Always confirm before writing.** Never add, update, or close a matter without showing the user the proposed timeline and getting explicit approval.
3. **Folder files and Gmail are supplementary, not authoritative.** The timeline is drafted from local files and emails but the user's corrections override everything.
4. **Timelines are append-only on update.** When merging, never delete or alter existing timeline entries. Only add new ones in chronological position.
5. **Find the tracker automatically.** See "Finding the Tracker" section below. If no tracker exists, create a fresh one in the Open Files directory.
6. **Preserve existing data.** When editing the tracker, never overwrite or delete existing rows. Only append or modify the targeted row.
7. **Flag if Gmail is unavailable.** If Gmail MCP tools are not available in the current environment, build the timeline from folder contents and tell the user. If both Gmail and folder are empty, proceed with manual entry — ask them to dictate the timeline events.
8. **Back up before writing.** Before any write operation (new, update, close), create a timestamped backup of the tracker (e.g., `matter-tracker-backup-2026-03-18.xlsx`) in the same directory. After writing the updated tracker, **verify the written file opens cleanly** by re-loading it with openpyxl and confirming the expected sheet and row count. Only after successful verification, delete older backup files (keep the most recent one). If verification fails, alert the user that the write may have corrupted the file and point them to the backup.
9. **Check for Excel lock files.** Before writing, check for a lock file (`~$matter-tracker.xlsx`) in the same directory. If found, warn the user: "The tracker appears to be open in Excel (`~$matter-tracker.xlsx` lock file detected). Close it in Excel before I write, or the save may fail or corrupt the file." Wait for confirmation before proceeding.
10. **Check for duplicates before adding.** Always run the Duplicate / Conflict Check before inserting a new matter row.
11. **Search deep for new matters.** Do not limit Gmail search to 90 days for new matters. The user may be retroactively adding long-running files. Paginate through results until you find the earliest correspondence, or the user tells you to stop.
12. **Always populate Next Action.** Every new, update, and close operation must set the Next Action / Deadline column. If no clear deadline exists, state the next procedural step.
13. **Gmail first, then folder scan.** Gmail provides the primary timeline backbone (communications, instructions, scheduling). The folder scan supplements it with document-level evidence (filed originals, endorsements, executed agreements).
14. **Read documents thoroughly.** Read every document that could contain a timeline event or client contact info. Always read engagement letters and intake forms (even if they seem purely administrative — they contain contact info). Skip only clear duplicates and purely administrative files (invoices, receipts).
15. **Scope the folder scan by operation.** For update/close, only process files modified since the Last Activity date — don't re-read the entire folder history. For new matters, scan everything.
16. **Handle multi-matter client folders.** If a client folder has subfolders for separate matters, scope the scan to the relevant subfolder. Match by matter description or keywords. If ambiguous, ask the user.
17. **Always populate column U (Other Parties).** On every new, update, and close, extract the names of all non-client, non-opposing parties from emails and folder files and write them to column U. This is critical for conflict check coverage.

## Finding the Tracker

The tracker file `matter-tracker.xlsx` lives in the Open Files directory alongside the matter folders. To find it:

1. **Check the current working directory** for `matter-tracker.xlsx`.
2. **If not found**, check the CWD's parent directory.
3. **If not found**, check one more level up (the grandparent directory).
4. **If still not found after three checks**, ask the user: "I can't find `matter-tracker.xlsx`. What directory is your Open Files folder in?" Do NOT glob recursively from the home directory — that's too slow on a large filesystem.
5. **If no tracker exists at all**, create a fresh one from the template (see Template Creation) in the directory the user specifies.

Once found, remember the tracker path for the rest of the session.

## Locating the Matter Folder for Briefs

When writing `_matter-brief.md` after an update or close, you need to resolve the Matter Folder name from column T to an actual path on disk. The tracker and matter folders live in the same parent directory (the Open Files directory).

**Primary approach**: The matter folder is a sibling directory of the tracker file. List the subdirectories in the same directory as `matter-tracker.xlsx` and find the one matching column T. Write the brief to `<open-files-dir>/<matter-folder>/_matter-brief.md`.

**Fallback — column T is blank**: List all sibling directories and do a case-insensitive fuzzy match against the client name (try last name, first name, entity name, and permutations). If a match is found, use it and populate column T for future use. If no match, ask the user for the folder path.

**Fallback — no match found**: Save the brief to the same directory as the tracker as `_matter-brief-[client-name].md` and tell the user to move it to the matter folder. Don't let the brief step fail silently — the user should know where it ended up.

## Matter Brief Format

When writing or refreshing `_matter-brief.md` in the matter folder (triggered by update or close operations), use this format. Keep it to **one page max** — this is a current-state snapshot for fast context loading in future sessions, not a full history.

```markdown
# [Client Name] — [File #]

## Matter Summary
[2-3 sentences: what this matter is about, who the parties are, what stage it's at]

## Key Terms / Provisions
[Only for transactional matters — price, term, material conditions, unusual clauses]

## Risks & Issues Flagged
- [Concise bullet points of flagged risks, unusual provisions, practical concerns]

## Positions Taken / Advice Given
- [Key advice given, positions taken in negotiations, strategic decisions made]

## Open Items
- [What's still unresolved, pending, or needs follow-up]

## Last Updated
[Date of this update]
```

Omit any section that doesn't apply (e.g., skip "Key Terms" for litigation). If the brief already exists, read it first and **replace stale content** rather than appending — the brief should always reflect the current state of the matter, not accumulate history. The timeline in the tracker spreadsheet handles the chronological record.

**Privilege warning**: The "Positions Taken / Advice Given" and "Risks & Issues Flagged" sections contain solicitor-client privileged content. The brief file (`_matter-brief.md`) should remain in the lawyer's internal file and **must not be shared with clients, opposing parties, or included in any document production**. Add the following header to every brief:

```
> PRIVILEGED & CONFIDENTIAL — Solicitor-Client Privilege / Work Product
```
