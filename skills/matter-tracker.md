---
name: matter-tracker
description: "Use this skill whenever the user says 'new matter [name]', 'update matter [name]', or 'close matter [name]'. Also trigger on: 'show my open files', 'what's open', 'matter list', 'file list', 'CRM', 'matter tracker', 'conflict check', 'limitation period', 'court deadline', or any reference to tracking client files, opening/closing/updating matters, checking for conflicts, tracking limitation periods, or reviewing the status of legal work. Trigger even if the phrasing is casual, e.g. 'new matter Smith', 'update matter Patel', 'close matter Jones', 'run a conflict on Lee', 'what's the limitation on the Effa file'. Also trigger when the user uploads a .xlsx file and references client matters, or asks to pull emails for a client to update their file. Do NOT trigger on 'let's work on [name]', 'pull up [name]', or 'where are we with [name]' — those belong to the work-on-matter skill for loading context. Always use this skill in combination with the xlsx skill for spreadsheet operations."
---

# Matter Tracker — Open Files CRM

## Overview

This skill maintains a spreadsheet-based CRM of the user's open legal matters. It supports three core operations:

1. **Add** a new matter (with Gmail and client folder auto-population)
2. **Update** an existing matter (add notes, update description, log activity)
3. **Close** a matter (set status to Closed, record close date)

Plus a **Review** mode to display current open matters in conversation.

## Dependencies

- **xlsx skill**: Use the xlsx skill (follow its workflow) for spreadsheet creation/editing. **If the xlsx skill is unavailable**, use openpyxl directly following the formatting rules in this skill — the schema, row formatting, and column width specifications below are self-sufficient for direct openpyxl operations.
- **Gmail MCP tools**: the connected Gmail MCP exposes `search_threads` (find threads by query) and `get_thread` (read a full thread by ID). There is no message-level search and no single-message read, everything is thread-level. `search_threads` truncates the per-thread message list, so treat it as a thread-ID discovery tool and re-read each thread with `get_thread` to see all of its messages. Use these directly, no loading step required. If Gmail tools are unavailable in the current environment, fall back to folder scan and manual entry.
- **Local file tools**: Use Glob, Grep, and Read tools to scan the client's matter folder for documents, correspondence, court filings, and other files that inform the timeline.
- **calendar-sync skill**: After every successful tracker write (new, update, close), invoke `calendar-sync` to push, update, or cancel deadline events on the Key Dates calendar. See "Calendar Sync Hooks" below and the `calendar-sync` SKILL.md for the call conventions. If calendar-sync or the Google Calendar MCP is unavailable, continue with the tracker write anyway — calendar sync should never block the tracker update.

## Conventions

- **"the lawyer"** in timeline entries refers to the user. Always use "the lawyer" as shorthand for the user in timeline entries — e.g., "the lawyer sent demand letter to opposing counsel."
- **"Client"** refers to the person/entity who retained the lawyer on the matter.

## Spreadsheet Schema

The tracker spreadsheet uses two sheets: **"Open Matters"** (active files) and **"Closed Matters"** (archived files). Both sheets share the same column schema:

### Sheet 1: "Open Matters" — all active files
### Sheet 2: "Closed Matters" — archived files moved here on close

Both sheets use these columns:

| Column | Header | Format | Notes |
|--------|--------|--------|-------|
| A | File # | Text (e.g. "2026-001") | Auto-assigned: YYYY-NNN. Next number = highest NNN across **both** the Open Matters AND Closed Matters sheets, + 1. A closed matter keeps its number forever — never reuse it. (Scanning only Open Matters reuses a number when a matter is closed shortly before a new one is opened.) |
| B | Client Name | Text | Primary client name. **Standard format: `Entity Name (Principal Name)`** — e.g. "Acme Group Inc. (Blake Murphy)". For individual clients with no entity, just use their name. For individuals acting through a numbered company, lead with the entity: "10014056 Holdings LLC (Jane D)". If multiple key individuals exist (e.g. two directors), comma-separate them inside the brackets: "ABC Real Estate Solutions Inc. (Bob Adams, Carol Chen)". **Never use slash format** (e.g. "Name / Corp") — always use the brackets format for consistency. This ensures the conflict check catches both the entity and the individual(s) behind it. |
| C | Matter Description | Text (wrap text) | Brief description of the engagement |
| D | Status | Text | "Open" or "Closed" |
| E | Date Opened | Date (YYYY-MM-DD) | Date the file was opened |
| F | Date Closed | Date (YYYY-MM-DD) | Blank until closed |
| G | Last Activity | Date (YYYY-MM-DD) | Updated on every interaction |
| H | Opposing Party | Text | If applicable; blank otherwise |
| I | Next Action / Deadline | Text (wrap text) | Key upcoming deadline or next step — see Next Action Format below |
| J | Timeline | Text (wrap text) | Concise chronological timeline built from Gmail and client folder files — see Timeline Format below |
| K | Client ID Verified | Text | "✓" once verified via Veriff; "Pending" if not yet done |
| L | Conflict Check Done | Text | "✓" once conflicts check completed; "Pending" if not yet done |
| M | Client Email | Text | Client's email address; blank if not yet collected |
| N | Client Phone | Text | Client's phone number; blank if not yet collected |
| O | Client Address | Text | Client's mailing address; blank if not yet collected |
| P | Discovery Date | Date (YYYY-MM-DD) | Date of discovery for limitation purposes — ONLY set when there is a live or potential claim. Leave blank for transactional/advisory matters. |
| Q | Limitation Statute | Text | Key identifying the applicable limitation statute (e.g. "limitations_act_basic", "human_rights"). ONLY set when there is a live or potential claim. Leave blank for transactional/advisory matters. |
| R | Limitation Deadline | Date (YYYY-MM-DD) | Calculated or manually entered limitation expiry. Auto-calculated from Discovery Date + Statute if both are set. ONLY set when there is a live or potential claim. |
| S | Court Deadlines | Text (JSON array) | Court-ordered deadlines stored as JSON. Each entry: {"date":"YYYY-MM-DD","description":"what's due","source":"endorsement or order reference"}. Only for bespoke deadlines from endorsements/orders — NOT routine rule-based deadlines like "defence due in 20 days". |
| T | Matter Folder | Text | Subfolder name (NOT a full path) within the Open Files directory (e.g. "Smith, J." — not "/Users/.../Smith, J."). Used to resolve the matter folder path on disk. When creating a new matter, search the Open Files directory for a subfolder matching the client — try ALL of: the individual's last name, first name, full name, "Last, First" format, the company/entity name, and common abbreviations. Client folders are often named after the entity rather than the person (e.g. "Summit Industries" not "Heaps, Toby"). Cast a wide net: list all subfolders and grep for each search term separately. Write just the subfolder name. Leave blank if no match. |
| U | Other Parties / Related Persons | Text (wrap text) | All non-client, non-opposing parties involved in the matter: co-plaintiffs, co-defendants, witnesses, guarantors, landlords, agents, process servers, adjusters, corporate officers, and anyone else whose name should trigger a conflict check. Comma-separated. Include individuals behind corporate opposing parties if known (e.g. if opposing party is "Acme Corp", and the director is "Jane Doe", list "Jane Doe" here). This column is searched during conflict checks to catch indirect conflicts. |
| V | Matter Type | Text | Free-text classification of the matter (e.g. "Litigation", "Solicitor", "Transactional", "Advisory", "Small Claims", "Demand Letter"). No fixed enum — keep values consistent with prior rows for filterability, but allow new categories as the practice evolves. Used for sorting and filtering matters in reports; not client-facing. |

### Formatting

- **Header row**: Bold, light blue fill (#D6E4F0), frozen pane, auto-filter enabled
- **Font**: Arial 10pt throughout
- **Column widths**: A=12, B=22, C=40, D=10, E=14, F=14, G=14, H=22, I=30, J=60, K=14, L=14, M=22, N=16, O=30, P=14, Q=30, R=14, S=40, T=50, U=50, V=18
- **Status column**: Use data validation (Open/Closed)
- **Limitation Statute column (Q)**: Use data validation — list = "limitations_act_basic,limitations_act_ultimate,cpa_2_year,employment_standards,human_rights,construction_act,insurance_act,municipal_liability,custom"
- **Wrap text** on columns C, I, J, S, and U **only** — do NOT set wrap_text on other columns

### Row Formatting (CRITICAL — must match existing rows exactly)

When appending or modifying any data row, **clone the formatting from the nearest existing data row** to ensure visual consistency. Specifically:

1. **Borders**: Every cell in columns A–V must have thin borders on all four sides (left, right, top, bottom). Use `Border(left=Side(style='thin'), right=Side(style='thin'), top=Side(style='thin'), bottom=Side(style='thin'))`.
2. **Wrap text**: Set `wrap_text=True` ONLY on columns C (Matter Description), I (Next Action / Deadline), J (Timeline), S (Court Deadlines), and U (Other Parties). All other columns must have `wrap_text=False` or no wrap setting.
3. **Font**: Arial 10pt, no bold (bold is header row only).
4. **Alignment**: Do not set vertical alignment to 'top' or any other value unless the existing rows use it. Match whatever the existing data rows use.

**Implementation pattern** (openpyxl):
```python
from openpyxl.styles import Font, Border, Side, Alignment

thin = Side(style='thin')
border = Border(left=thin, right=thin, top=thin, bottom=thin)
font = Font(name='Arial', size=10)
wrap_cols = {3, 9, 10, 19, 21}  # C, I, J, S, and U only (V does not wrap)

for col in range(1, 23):  # A through V
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

The Timeline column (J) is the core deliverable of every command. It is a concise, chronological log of the matter built from Gmail correspondence and client folder files. Format:

```
YYYY-MM-DD: [one-line summary of what happened]
YYYY-MM-DD: [next event]
YYYY-MM-DD: [next event]
```

Rules for the timeline:
- One line per significant event (email sent, document received, call referenced, deadline set, etc.)
- Chronological order, oldest first
- Each line is date + colon + short plain-language summary (no legalese, no fluff)
- Keep each entry to ~10-15 words max
- Include who did what where relevant (e.g. "Client sent signed SPA to opposing counsel")
- Use "the lawyer" to refer to the lawyer — e.g. "the lawyer sent demand letter"
- **Always include the intake/retention event** as the first timeline entry (e.g. "Client retained the lawyer re: ...")
- **Always include filing events** (e.g. "the lawyer sent claim to process server for filing and service")
- If closing, append: `YYYY-MM-DD: FILE CLOSED.`
- On update, merge new entries into the existing timeline in chronological order — never duplicate or delete existing entries
- **Long timelines**: Never compress or summarize timeline entries — keep every entry at full granularity regardless of length. If the cell gets unwieldy in Excel, that's acceptable; the complete record is more valuable than a tidy spreadsheet.

Example:
```
2026-01-15: Initial client intake call re: share purchase
2026-01-18: Sent engagement letter to client
2026-01-22: Received draft SPA from opposing counsel (Davies LLP)
2026-02-01: Sent markup of SPA to opposing counsel
2026-02-10: Client approved final SPA
2026-02-14: Closing — executed SPA exchanged
```

## Audit Columns (K–L)

Columns K and L are simple compliance checkmarks set on every new matter:

- **K — Client ID Verified**: Default to "Pending" on file opening. ID verification is performed via **Veriff** (the lawyer sends the client the Veriff link manually by email, outside Cowork). Set to "✓" only once the client has completed Veriff and the lawyer confirms it. The checkmark should reflect a verification that actually happened (LSO By-Law 7.1 client identification and verification), not a file-open default. If verification is still outstanding when the file is opened, "Pending" is the accurate state.
- **L — Conflict Check Done**: Set to "✓" on file opening. Confirms a conflicts check was completed before the file was opened. Default to "✓" for all new matters.

Column L (Conflict Check Done) is set to "✓" automatically when a new matter is created, since the conflicts check is run as part of opening the file, and is carried over on close. Column K (Client ID Verified) defaults to "Pending" and flips to "✓" only when Veriff is actually confirmed, so it is not auto-checked. Neither column requires Gmail searches.

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
- **Q — Limitation Statute**: One of: `limitations_act_basic` (2yr), `limitations_act_ultimate` (15yr), `cpa_2_year` (2yr), `employment_standards` (2yr), `human_rights` (1yr), `construction_act` (2yr), `insurance_act` (1yr), `municipal_liability` (10-day notice + 2yr), or `custom`. If none of the named statutes apply, use `custom` and ask the user to provide the deadline manually.

  **Two presets carry traps — confirm with the user before relying on them, and surface the caveat in the confirmation message:**
  - `insurance_act` (1yr) fits only specific claim types (e.g. auto property under s. 259.1). Most insurance claims are governed by the 2-year Limitations Act period, and SABS/LAT disputes run their own 2-year clock — for those, `limitations_act_basic` or `custom` is the right key. Ask which period applies; never default to 1 year just because the matter involves an insurer.
  - `construction_act` (2yr) captures only the action limitation. The deadlines that actually kill construction files — 60-day lien preservation and 90-day perfection — are NOT generated by this preset. Track them explicitly in column S or column I so they reach the calendar.
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

**Format**: Comma-separated names. Include role context where helpful: "Alex Brooks (co-plaintiff/witness), Dana Reyes (opposing counsel, Reyes Law)".

**Why this matters**: The conflict check searches this column. If a future prospective client's name appears here, the user is alerted before opening a file that could create a conflict.

## Core Research Procedure

**Every command (new, update, close) begins with a Gmail pull, then a client folder scan.** These are the universal first steps:

### Step A — Gmail Search

Gmail provides the primary chronological backbone of the timeline — it captures communications, instructions, scheduling, and references to key events.

**The Gmail pull is two-pass when sender addresses are known.** Each pass catches what the other misses, and both run when the inputs exist. The reason for two passes: Gmail's message search matches each message individually against the query, so a one-line reply on an old thread ("got it, will sign tomorrow") carries no matter-specific keywords and a keyword-only pass will silently miss it. The from-address pass is the safety net for those short replies.

1. Use the Gmail MCP tools directly (`search_threads` to find threads, `get_thread` to read them).

2. **Pass A — Keyword pass.** Build the query from these inputs, joined with OR:
   - **Client name** from the matter row (entity AND principal name in brackets, as separate OR'd terms).
   - **Opposing party** from column H if known.
   - **Named role-holders** from any existing `_matter-brief.md` (`## Roles` section): opposing principals, opposing counsel, named witnesses, experts, agents, paralegals — parse names out of the role lines, skip the client (already covered) and skip generic role labels like "Landlord" or "counsel". The reason: a third party who isn't on file as a known sender (e.g., the client forwarding correspondence, a paralegal at the opposing firm using a generic firm address) may mention the matter by referring to one of these named players. Pass B will not catch them; Pass A only catches them if the role-holder's name is in the keyword list.
   - **Matter-specific keywords** from column C that are unusual enough to be useful (e.g., property address, court file number, distinctive entity name). Don't pad with generic terms ("lease", "claim", "demand letter") — they over-match.

3. **Pass B — From-address pass.** Catches short replies on existing threads where the message body contains no matter-specific text. Build the query as `(from:addr1 OR from:addr2 OR ...) newer_than:Xd` using:
   - **Client Email** from tracker column M.
   - **Email addresses parsed out of `_matter-brief.md` `## Roles`** (opposing counsel, opposing principals, third-party addresses).
   - **Court / tribunal / third-party addresses** that have appeared on this matter before (look in prior comms file or earlier timeline entries).

   Pass B is **skipped** for new-matter operations where no addresses are known yet (column M blank, no brief, no prior comms). The keyword pass alone is acceptable in that case because there is no thread history to silently slip through. For **update** and **close** operations, Pass B is non-negotiable when any addresses are on file — without it, short replies on old threads silently slip through.

4. **Time windows:**
   - For **new matter**: use **no time limit** by default. Paginate through results to find the earliest correspondence. If results exceed 3 pages, ask the user: "I'm finding emails going back to [date]. Should I keep going deeper or is that far enough?" Pass B is generally skipped here because no addresses are known yet.
   - For **update matter**: search from the Last Activity date onward on both passes.
   - For **close matter**: search from the Last Activity date onward on both passes.

5. **Combine and dedupe.** Union the unique THREAD IDs returned by Pass A and Pass B — not the message lists, which `search_threads` truncates (see Dependencies). A thread that hits both passes is fine; process it once. The full message list comes from `get_thread` in the next step.

6. Read email threads until the timeline is complete. **Use `get_thread` on each thread** — do not rely on snippets alone, as snippets truncate critical details like dates, times, locations, and court file numbers. Read every thread that could contain a timeline event. There is no cap on how many threads to read — the goal is a complete chronological record. If there are dozens of threads, read them all. For efficiency, prioritize court/scheduling emails first, then substantive correspondence, then administrative emails — but do not skip threads just because there are many.
7. Extract: dates, key actions, parties involved, documents exchanged, deadlines, outcomes.
8. **Extract client contact info**: On every Gmail pull, look for the client's email address (from the "From" header), phone number, and mailing address (from email signatures, body text, or attached documents). If found and columns M-O are blank, populate them.
9. **Court and scheduling emails are highest priority.** When any email originates from a court address (e.g. @ontario.ca, @scj-csj.ca, any court clerk) or references a court file number, scheduling, or hearing date, **always read that message in full** and extract all dates, times, locations, and Zoom/video links. These must be captured verbatim in the Next Action field (with exact date and time) and in the Timeline. Never summarize or skip a court scheduling email.
10. Build the base timeline from Gmail results.

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

Before adding a new matter, **always run a full conflicts check**. This has three parts: a duplicate check against the tracker (same client), an adverse interest check against the tracker (cross-party conflicts), and a beyond-the-tracker check (folder names on disk + Gmail history) for files that predate the tracker.

### Part 1 — Duplicate Check (same client name)

1. Load the tracker (see "Finding the Tracker" below).
2. Search the "Open Matters" sheet for the client name (case-insensitive partial match on column B — Client Name).
3. Also check "Closed Matters" if the sheet exists.
4. If a match is found:
   - If on Open Matters: **stop and ask** — "There is already an open file for [Name] (File #[X]). Did you mean to update that file, or is this a separate matter?"
   - If on Closed Matters only: **flag but proceed** — "Note: [Name] had a previously closed file (File #[X]). Opening a new file."

### Part 2 — Adverse Interest Check (cross-party conflicts)

5. If the new matter has an opposing party, search **all rows on both sheets** for that opposing party name in columns B (Client Name), C (Matter Description), and U (Other Parties). This catches the case where someone you're suing (or negotiating against) is an existing or former client, or was involved in another matter. Also check column H for the same name — a hit there is NOT a conflict (being adverse to the same party twice is fine), but report it as repeat-litigant intelligence: prior dealings with this opponent are worth knowing at intake.
6. Also search columns H (Opposing Party) and U (Other Parties) across all rows for the **new client's name**. This catches the case where the new client was previously on the other side of one of your matters.
7. Also search column C (Matter Description) for the new client's name — descriptions sometimes mention parties not captured elsewhere.
8. If any adverse match is found: **stop immediately and alert the user** — "Potential conflict: [New Opposing Party] appears as a client in File #[X], or [New Client] appears as an opposing party in File #[X]. You must resolve this conflict before opening this file."
9. If no matches on either part, proceed to Part 3.

### Part 3 — Beyond the Tracker (filesystem + Gmail)

The tracker only knows about matters that were entered into it. Files that predate the tracker leave two other traces: a folder on disk and an email trail. Checking both brings the open-file conflict check up to the standard work-on-matter already imposes on mere *statements* about prior involvement (its Prior-Matter Fact Discipline) — and opening a file is the higher-stakes act. Column L's "✓" should certify a check that actually looked everywhere.

10. **Folder grep**: list the Open Files directory's subfolder names (`ls -1 > /tmp/dirlist.txt`, then case-insensitive grep — same technique as the column T search) for the new client's name AND the opposing party's name (last name, first name, entity name, permutations). A folder hit means a file existed even if the tracker has no row for it.
11. **Gmail search**: run `search_threads` on the new client's name and the opposing party's name (plus email addresses if known), no time limit. Old retainers leave email trails.
12. Any hit from either source: surface what was found and where, and wait for the user's direction before opening the file. All three sources clear → proceed normally.

**Search scope summary**:
- New client's name → columns B (duplicate check), C, H, and U — both sheets
- New opposing party's name → columns B, C, and U — both sheets (an H-on-H hit is repeat-litigant intel, not a conflict — report it but don't block)
- Newly discovered Other Parties → columns B, C, H, and U — both sheets (the post-research re-check in NEW MATTER step 5)
- Part 3 → Open Files folder names and Gmail history, for both the new client and the opposing party

## Workflows

### 1. NEW MATTER

**Trigger**: "new matter [name]" or "new matter"

**Steps**:

1. Extract the client name from the command. If absent, ask.
2. **Run the Duplicate / Conflict Check** (all three parts — tracker, folder names, Gmail).
3. **Run the Core Research Procedure** (folder scan + Gmail, no time limit — paginate to find full history). **Both steps (Gmail search AND folder scan) must be completed before drafting the timeline.** Do not skip the folder scan — even if Gmail provides a rich history, the folder often contains court documents, endorsements, and filed originals that Gmail misses. The folder scan also confirms the Matter Folder name for column T.
4. From the folder files and emails, draft:
   - Client Name — **use the standard format: `Entity Name (Principal Name)`**. If the client is a corporation or other entity, identify the principal/directing mind from the correspondence or folder files. If you can't identify the principal from the documents, ask the user: "Who is the principal/directing mind of [entity]?" For individual clients with no entity, just use their name.
   - Matter Description (one-line summary of the engagement)
   - Opposing Party (if identifiable)
   - Next Action / Deadline (the most critical upcoming step)
   - Timeline (full chronological log from folder files and emails — include retention/intake and filing events)
   - Other Parties (anyone else involved — co-parties, witnesses, lawyers, agents)
   - Client Email / Phone / Address (if found in emails or folder files)
5. **Re-run the adverse-interest check on newly discovered names.** The step-2 conflict check ran BEFORE the research pull, so it never saw the names that just landed in Other Parties — guarantors, co-defendants, witnesses, directors behind opposing entities. Search columns B, C, H, and U on both sheets for each newly identified name. Any hit: stop and alert the user exactly as in the main conflict check. This closes the gap where a conflict walks in through a party discovered mid-research.
6. **Present to user for confirmation:**
   ```
   New matter for [Name]:
   Matter: [description]
   Opposing: [party or "N/A"]
   Other Parties: [list or "None identified"]
   Next Action: [deadline or next step]
   Contact: [email] | [phone] | [address] (or "not found" for each)
   Timeline:
   YYYY-MM-DD: [event]
   YYYY-MM-DD: [event]
   ...

   Add to tracker? Any corrections?
   ```
7. After confirmation:
   - Load the tracker (see "Finding the Tracker"), assign next File # (max NNN across BOTH Open and Closed Matters sheets, + 1 — never reuse a closed matter's number), append row
   - If no tracker exists -> create new tracker from template in the Open Files directory, then add row
   - Set Status = "Open", Date Opened = earliest timeline date (or today), Last Activity = today
   - Set Conflict Check Done = "✓" (the conflicts check was run as part of opening). Set Client ID Verified = "Pending" unless the lawyer confirms Veriff is already complete for this client.
   - Populate Client Email/Phone/Address (columns M-O) if available from folder files or emails
   - Populate Other Parties (column U) with all non-client, non-opposing parties identified
   - **If the matter involves a claim**: ask about discovery date and limitation statute; populate columns P-R. Flag if limitation is within 6 months.
   - **If the matter is transactional/advisory**: leave columns P-R blank.
   - Leave Court Deadlines (S) blank unless folder files or emails reveal a specific court-ordered deadline.
   - **Matter Folder (T)**: Search the workspace directory for a subfolder matching the client. **All matching must be case-insensitive.** First, dump the full directory listing to a text file using `ls -1 > /tmp/dirlist.txt`, then grep against that file — this avoids shell issues with special characters (colons, ampersands, parentheses, etc.) in folder names. Try matching against ALL of these permutations of the client name: "First Last" (e.g. "wayne evans"), "Last, First" (e.g. "Taylor, Wayne"), "Last First" (no comma), just the last name, just the first name, and any company/entity name from the matter description or opposing party field. **Also search for the mother/father/third-party name if the matter is brought on someone else's behalf** (e.g. for "Reed v. Blake" brought by Patricia Moore on behalf of June Reed, search for "Moore", "Patricia", "Reed", and "June"). Folders are often named in lowercase or informal formats (e.g. "wayne evans" not "Taylor, Wayne"), or after the entity rather than the person (e.g. "Summit Industries" not "Heaps, Toby"), and frequently contain special characters like colons (e.g. "Patricia Moore : Reed"). Cast a wide net — grep each search term separately and case-insensitively against the text file listing. If found, write just the subfolder name exactly as it appears on disk. If not found, leave blank.
8. Save the updated tracker to disk.
9. **Calendar sync**: Invoke the `calendar-sync` skill's `reconcile(new_row)` for this matter. This pushes any limitation date (column R), court deadlines (column S), and dated Next Action (column I) to the Key Dates calendar with the appropriate reminder schedules. Report back to the user: "Pushed N events to Key Dates." If calendar-sync is unavailable, skip this step and note it once — do not block the tracker write.

### 2. UPDATE MATTER

**Trigger**: "update matter [name]" or "update matter"

**Steps**:

1. Extract the client name. If absent, ask.
2. Load the tracker (see "Finding the Tracker"). Find matching row (case-insensitive partial match; if ambiguous, ask).
3. **Run the Core Research Procedure** (folder scan + Gmail from Last Activity date onward).
4. Read the existing Timeline from the spreadsheet.
5. Merge new events into the existing timeline in chronological order. Do not duplicate or remove existing entries.
6. **Present the updated timeline to user for confirmation:**
   ```
   Updated timeline for [Name]:
   [existing entries]
   [NEW] YYYY-MM-DD: [new event]
   [NEW] YYYY-MM-DD: [new event]

   Next Action: [updated next step/deadline]
   Also update matter description? Currently: "[current]"
   Confirm?
   ```
7. After confirmation:
   - Write merged timeline to the Timeline column
   - Update Last Activity to today
   - Update Next Action / Deadline
   - Update Matter Description if scope changed
   - Update Client Email/Phone/Address if new contact info found in folder files or emails
   - Update Other Parties (column U) if new parties were identified
   - If folder files or emails reveal a new court date, endorsement, or order with a deadline, add it to the Court Deadlines JSON (column S) and alert the user
   - **Expired court deadlines — confirm before removing.** A passed date does not mean the obligation was satisfied; it may have been MISSED, which is exactly what most needs surfacing. For each column S entry whose date has passed, check the research pull for evidence it was satisfied (filing confirmation, email, endorsement). Evidence found: remove it and tell the user ("Cleared: 2026-02-15 — Amend claim — amended claim filed Feb 14"). No evidence: do NOT remove — flag it ("2026-02-15 — Amend claim: deadline passed, no evidence it was done. Was this handled?") and wait for the answer. Silent removal would also gut the overdue-triage skill, whose job is to investigate exactly these entries.
   - If the matter now involves a claim but limitation columns (P-R) are blank, flag this and ask the user about discovery date and statute
   - **If column T (Matter Folder) is blank**, attempt to populate it now using the folder resolution logic from the NEW MATTER workflow.
8. Save the updated tracker to disk.
9. **Write/refresh `_matter-brief.md`** in the matter folder (column T). If the brief exists, read it and update with current information. If it doesn't exist, create it. The brief is a current-state snapshot of the matter (soft warning at 250 lines, no hard cap — see "Matter Brief Format" below). This step happens automatically after saving the tracker — no need to ask the user for separate confirmation.
10. **Calendar sync**: Invoke `calendar-sync.reconcile(updated_row)`. This creates new events for any newly-added deadlines, updates events whose dates or descriptions changed, and deletes events for deadlines that were pruned (e.g., expired court deadlines removed from column S). Report the diff to the user: "Calendar sync: X added, Y updated, Z removed." If any events were deleted because a deadline passed, name them explicitly so the user sees what's no longer on the calendar.

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
11. **Calendar sync**: Invoke `calendar-sync.cancel_all_for_matter(file_number)` to remove every event on Key Dates for this file — court, limitation, follow-ups, and any third-party prompts. Confirm: "Cancelled N events on Key Dates." Closed files should leave no trace on the calendar.

### 4. REVIEW OPEN MATTERS

**Trigger**: "show my open files", "what's open", "matter list", "file summary"

**Steps**:

1. Load tracker.
2. Read the "Open Matters" sheet.
3. Display a clean summary in conversation:

```
You have X open matters:

1. 2026-001 | Smith, J. | Share purchase agreement | Next: Awaiting signed docs | Last: 2026-03-01
2. 2026-002 | Patel, R. | Damage deposit — Small Claims | Next: 2026-04-15 Settlement conf. | Last: 2026-03-10
...
```

4. Flag any upcoming deadlines within the next 30 days (court deadlines from column S and Next Action dates from column I).
5. **Flag any limitation periods within 6 months** (column R). Limitation deadlines are the highest-priority alerts — display them prominently, e.g.: "LIMITATION: File #2026-002 (Patel) — limitation expires 2026-06-15 (89 days)."
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
3. Run Parts 1 and 2 of the Duplicate / Conflict Check (see above), treating the provided name as both a potential client name AND a potential opposing party name:
   - Search column B (Client Name) on both sheets for the name.
   - Search column C (Matter Description) on both sheets for the name.
   - Search column H (Opposing Party) on both sheets for the name.
   - Search column U (Other Parties) on both sheets for the name.
4. Run Part 3 of the conflict check (folder-name grep + Gmail search) on the name — pre-tracker files leave traces the spreadsheet can't show.
5. Report results clearly:

```
Conflict check for "[Name]":

[If matches found:]
  - File #2026-001: [Name] is the CLIENT (matter: [description], status: [open/closed])
  - File #2026-003: [Name] is the OPPOSING PARTY (client: [client name], matter: [description], status: [open/closed])
  - File #2026-005: [Name] appears in OTHER PARTIES (client: [client name], matter: [description], role: [if known])

[If no matches:]
  No conflicts found. "[Name]" does not appear as a client, opposing party, or related person in any open or closed matter, matter folder name, or email history.
```

6. This workflow is read-only — it does not modify the tracker.

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
6. **Calendar sync — verify stale deadlines first.** The row's column S entries and limitation date are as old as the close. Reconcile already skips past dates on its own, but a FUTURE-dated entry from before the close may no longer be real — confirm those with the user before pushing. Then invoke `calendar-sync.reconcile(reopened_row)`. If the user runs "update matter" right after reopening, the refreshed deadlines are picked up automatically.
7. Suggest running "update matter [name]" to do a fresh Gmail + folder scan and rebuild the timeline.

## Calendar Sync Hooks

Every write to the tracker fires a calendar-sync call. The goal: the Key Dates calendar is always a faithful projection of what's in the tracker.

**One-way sync only.** Tracker is the source of truth; calendar events are derived. Never read calendar events back into the tracker.

**Four deadline categories**, each with a distinct reminder schedule:
- **Court deadlines** (column S entries): 14 / 7 / 2 / 0 days before
- **Limitation periods** (column R): 60 / 30 / 14 / 7 / 0 days before
- **Client follow-ups** (dated entry in column I when not already a court/limitation date): 2 / 0 days before
- **Third-party follow-ups** (added ad-hoc by work-on-matter): 2 / 0 days before

**Reconciliation is idempotent.** Running `reconcile` twice in a row should produce no net changes. The calendar-sync skill handles dedup, date drift, and pruning of orphaned events.

**Report back to the user** after every reconcile call. Even a single-line summary ("Calendar sync: 2 added, 1 updated") matters — the user needs to know deadlines landed. Silent syncs erode trust in the system.

**When calendar-sync fails**, log the failure and continue. Never roll back a tracker write because calendar sync errored. The tracker is the record of truth; the calendar is convenience.

See `calendar-sync/SKILL.md` for the full spec, including event title format, sync-key convention, and the `reconcile`, `upsert_deadline`, `cancel_deadline`, and `cancel_all_for_matter` operations.

## Template Creation

When creating a new tracker from scratch, use openpyxl to build the spreadsheet with:

**Sheet 1 — "Open Matters":**
- Header row with formatting per the schema above (columns A-V)
- Freeze panes at row 2
- Auto-filter on the header row
- Data validation on column D (Status): list = "Open,Closed"
- Data validation on column Q (Limitation Statute): list = "limitations_act_basic,limitations_act_ultimate,cpa_2_year,employment_standards,human_rights,construction_act,insurance_act,municipal_liability,custom"
- Print area and page setup: landscape, fit to 1 page wide

**Sheet 2 — "Closed Matters":**
- Same header row formatting as "Open Matters" (bold, light blue fill #D6E4F0, same column widths, frozen pane, auto-filter)
- Same column schema (A-V)
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
8. **Back up before writing.** Before any write operation (new, update, close), create a timestamped backup of the tracker (e.g., `matter-tracker-backup-2026-03-18.xlsx`) in a `backups/` subfolder alongside the tracker. Create the `backups/` folder if it doesn't exist. After writing the updated tracker, **verify the written file opens cleanly** by re-loading it with openpyxl and confirming the expected sheet and row count. Do NOT delete older backups -- let the folder accumulate history. The user can prune manually if it ever grows too large. If verification fails, alert the user that the write may have corrupted the file and point them to the most recent backup in `backups/`.
9. **Check for Excel lock files.** Before writing, check for a lock file (`~$matter-tracker.xlsx`) in the same directory. If found, warn the user: "The tracker appears to be open in Excel (`~$matter-tracker.xlsx` lock file detected). Close it in Excel before I write, or the save may fail or corrupt the file." Wait for confirmation before proceeding.
10. **Check for duplicates before adding.** Always run the Duplicate / Conflict Check before inserting a new matter row.
11. **Search deep for new matters.** Do not limit Gmail search to 90 days for new matters. The user may be retroactively adding long-running files. Paginate through results until you find the earliest correspondence, or the user tells you to stop.
12. **Always populate Next Action.** Every new, update, and close operation must set the Next Action / Deadline column. If no clear deadline exists, state the next procedural step.
13. **Gmail first, then folder scan.** Gmail provides the primary timeline backbone (communications, instructions, scheduling). The folder scan supplements it with document-level evidence (filed originals, endorsements, executed agreements).
14. **Read documents thoroughly.** Read every document that could contain a timeline event or client contact info. Always read engagement letters and intake forms (even if they seem purely administrative — they contain contact info). Skip only clear duplicates and purely administrative files (invoices, receipts).
15. **Scope the folder scan by operation.** For update/close, only process files modified since the Last Activity date — don't re-read the entire folder history. For new matters, scan everything.
16. **Handle multi-matter client folders.** If a client folder has subfolders for separate matters, scope the scan to the relevant subfolder. Match by matter description or keywords. If ambiguous, ask the user.
17. **Always populate column U (Other Parties).** On every new, update, and close, extract the names of all non-client, non-opposing parties from emails and folder files and write them to column U. This is critical for conflict check coverage.
18. **Calendar sync runs after every tracker write.** New, update, and close all invoke the `calendar-sync` skill — new/update/reopen call `reconcile`, close calls `cancel_all_for_matter`. This is non-negotiable; the value of the tracker is undermined if its deadlines don't appear on the calendar. If calendar-sync fails, the tracker write still commits and the failure is logged to the user.

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

**Primary approach**: The client folder is a sibling directory of the tracker file. List the subdirectories in the same directory as `matter-tracker.xlsx` and find the one matching column T. **Then resolve the matter-specific subfolder, exactly as work-on-matter Step 2 does**: list the client folder's immediate contents; if a subfolder matches the matter description (column C) or opposing party (column H) by keyword, that subfolder is the matter folder and the brief belongs there; if no subfolder matches but `_matter-brief.md` already exists at the client folder's top level, use the top level; if subfolders exist, none match, and there is no top-level brief, ask the user which subfolder. Write the brief to the resolved folder. The reason this matters: writing at the client folder's top level when the work lives in a subfolder is how a multi-matter client ends up with two divergent briefs — or has matter A's brief overwritten with matter B's content.

**Fallback — column T is blank**: List all sibling directories and do a case-insensitive fuzzy match against the client name (try last name, first name, entity name, and permutations). If a match is found, use it and populate column T for future use. If no match, ask the user for the folder path.

**Fallback — no match found**: Save the brief to the same directory as the tracker as `_matter-brief-[client-name].md` and tell the user to move it to the matter folder. Don't let the brief step fail silently — the user should know where it ended up.

## Matter Brief Format

When writing or refreshing the matter files (triggered by update or close operations), use the three-file architecture defined in the work-on-matter skill: `_matter-brief.md` (snapshot — live sections replaceable, no hard cap; superseded items demote to `## Resolved / Historical`), `_matter-decisions.md` (append-only log of strategic reasoning, no cap), and `_matter-comms.md` (append-only log of file-specific operational rules, no cap). All three live in the matter folder.

**This skill's update workflow primarily writes the brief.** Update and close operations rebuild the brief from a fresh research pull. The decisions log and comms preferences are owned by the work-on-matter skill — this skill should never overwrite, edit, or prune them. If those files exist in the matter folder, leave them alone. (The exception: if the close operation surfaces a final closeout note that future sessions should be able to find, append it to the decisions log as a new entry, never edit existing entries.)

**Brief length: soft warning at 250 lines, no hard cap.** If a save would push the brief past 250 lines, surface it to the user before writing: "Brief is at [N] lines. Want me to refactor (demote superseded Risks/Positions/Open Items to `## Resolved / Historical`, move reasoning to _matter-decisions.md) or save as-is?" Wait for direction. Do NOT auto-refactor — the model can't reliably tell which items are still live versus superseded, and a silent change can drop content that should have been kept. Demote, never delete: same semantics as the work-on-matter skill.

If the content pushing the brief long is reasoning rather than current state, propose moving it to `_matter-decisions.md` as part of the refactor offer.

This 250-line soft warning matches the work-on-matter skill so briefs produced by either skill are sized consistently.

**Brief format (`_matter-brief.md`):**

```markdown
> PRIVILEGED & CONFIDENTIAL — Solicitor-Client Privilege / Work Product

# [Client Name] — [File #]

## Matter Summary
[2-3 sentences: what this matter is about, who the parties are, what stage it's at]

## Roles
- Name (role) — source: [email date / doc filename + page / tracker col X]
- Name (role) — source: [...]

## Key Terms / Provisions
[Only for transactional matters — price, term, material conditions, unusual clauses]

## Risks & Issues Flagged
- [Concise bullet points of flagged risks, unusual provisions, practical concerns]

## Positions Taken / Advice Given
- [Key advice given, positions taken in negotiations, strategic decisions made — current state, not historical reasoning. Reasoning lives in _matter-decisions.md.]

## Open Items
- [What's still unresolved, pending, or needs follow-up]

## Tracked Threads
- [Gmail thread ID] — "[short subject label]" — last seen YYYY-MM-DD

## Resolved / Historical
- [YYYY-MM-DD — Demoted item with one-line resolution note. Append-only.]

## Last Updated
[Date of this update]
```

Omit any section that doesn't apply (e.g., skip "Key Terms" for a litigation matter), except the Roles block, which is mandatory for every brief.

**Backup before every brief write.** Copy the existing `_matter-brief.md` (if any) to `backups/_matter-brief.YYYY-MM-DD.md` in a `backups/` subfolder of the matter folder. One backup per day; same-day overwrites fine. Verify the file re-opens cleanly after write; on failure, point the user to the most recent backup.

**Decisions log and comms preferences format and lifecycle:** Defined in the work-on-matter SKILL.md ("Decisions Log Format" and "Comms / Client Preferences Format" sections). This skill does not create or edit those files except as noted above for closeout notes.

**The Roles block is mandatory.** Every named party in the matter with their role and a source citation for that role. This is the single place in the brief where each person is pinned to a source. Paraphrasing a role in an outgoing email without first confirming it here is how role errors leak into client-facing work. Example:

```
## Roles
- Ben Mercer (Landlord principal, Metro Fashions Ltd.) — source: Lease Assignment executed Apr 7 2026, recital A
- Frank West (counsel for Assignor / Seller, 1234567 Ontario Inc.) — source: signature block of Lease Assignment; confirmed Apr 2 14:04 ET email
- Rita Moss (Metro Controller) — source: Apr 17 11:39 email re security deposit wire
- KRB Lawyers Inc. (counsel of record for Landlord) — source: s.2.8.1(c) of Lease Assignment
```

**Source tagging in the body.** Factual claims in the body sections (Risks, Positions, Open Items, Key Terms) follow this convention:

- Unmarked statement → read directly from a source document on file
- `[inferred]` → derived from other facts, not directly verified. Flags a claim that reads like a fact but is actually a deduction
- `[per client, unverified]` → stated by client in writing or on a call but not backed by a document
- `[TBC]` → to-be-confirmed; known to need a source

Use the tags sparingly but honestly. An untagged claim is a guarantee to the next session (and to the lawyer) that it came from a source you actually read. The most common failure this prevents: a mathematical inference or memory reach that reads as a fact and then lands in client-facing work.

**If the brief already exists**, read it first and **replace stale content** rather than appending — the brief should always reflect the current state of the matter, not accumulate history. **Two sections are exempt and must survive every rebuild: `## Tracked Threads` and `## Resolved / Historical`.** Tracked Threads is the persistence layer for work-on-matter's Pass C email refresh — wiping it silently breaks new-mail detection on the next session. Carry both sections over verbatim, then MERGE into Tracked Threads: add a line for every thread this operation's Gmail pull touched (thread ID, short subject label, latest message date as "last seen") and advance "last seen" on existing entries the pull re-read. The deep Gmail pull in this skill is the best seeding Tracked Threads ever gets — use it. When this skill resolves or supersedes a brief item, demote it to `## Resolved / Historical` with a dated one-line note instead of deleting it. The timeline in the tracker spreadsheet handles the chronological record. The `_matter-decisions.md` and `_matter-comms.md` files are owned by work-on-matter and must not be touched by this skill (except as noted above for closeout entries) — losing an entry from them is a memory leak the next session can't recover.

**Privilege warning**: The "Positions Taken / Advice Given" and "Risks & Issues Flagged" sections contain solicitor-client privileged content. The brief file (`_matter-brief.md`) should remain in the lawyer's internal file and **must not be shared with clients, opposing parties, or included in any document production**. Add the following header to every brief:

```
> PRIVILEGED & CONFIDENTIAL — Solicitor-Client Privilege / Work Product
```
