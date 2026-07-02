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

- **xlsx skill**: Use the xlsx skill (follow its workflow) for reading the tracker and for template creation. **If the xlsx skill is unavailable**, use openpyxl directly for reads — the schema summary below plus REFERENCE.md § Spreadsheet Schema are self-sufficient. All writes to an existing tracker go through `tracker_write.py` (see "Tracker Writes" below), never ad-hoc openpyxl.
- **Gmail MCP tools**: the connected Gmail MCP exposes `search_threads` (find threads by query) and `get_thread` (read a full thread by ID). There is no message-level search and no single-message read, everything is thread-level. `search_threads` truncates the per-thread message list, so treat it as a thread-ID discovery tool and re-read each thread with `get_thread` to see all of its messages. Use these directly, no loading step required. If Gmail tools are unavailable in the current environment, fall back to folder scan and manual entry.
- **Local file tools**: Use Glob, Grep, and Read tools to scan the client's matter folder for documents, correspondence, court filings, and other files that inform the timeline.
- **calendar-sync skill**: After every successful tracker write (new, update, close), invoke `calendar-sync` to push, update, or cancel deadline events on the Key Dates calendar. See "Calendar Sync Hooks" below and the `calendar-sync` SKILL.md for the call conventions. If calendar-sync or the Google Calendar MCP is unavailable, continue with the tracker write anyway — calendar sync should never block the tracker update.

## Tracker Writes — tracker_write.py

ALL tracker writes go through the write-guard CLI — never ad-hoc openpyxl:

```
python3 scripts/tracker_write.py <subcommand> --tracker "<tracker path>" ...
```

Subcommands this skill uses:

- `new-matter --client "..." --description "..." [--opposing --email --phone --matter-type --matter-folder --date-opened]` — appends the row, assigns the next File # (scanning BOTH sheets), sets Status = "Open", Date Opened, Conflict Check = "✓", Client ID Verified = "Pending"
- `update --file-no N --set "COLUMN=value" [--set ...]` — cell writes; multiple `--set` per call
- `timeline --file-no N --date YYYY-MM-DD --text "..."` — appends a Timeline entry and bumps Last Activity per the max rule
- `close --file-no N` / `reopen --file-no N` — sheet moves, keeping Status / Date Closed / Timeline consistent
- `court-deadline add --file-no N (--date YYYY-MM-DD | --anchor "trial date" --offset-days -30) --description "..." [--source "..."]` · `court-deadline remove --file-no N --index I` · `court-deadline resolve --file-no N --index I --date YYYY-MM-DD`
- Flags: `--dry-run`, `--json`, `--force-unlocked`

Every call does the Excel-lock check, a timestamped backup into `backups/`, an atomic save, and runs validate_tracker.py automatically (exit 3 + backup path on FAIL). **Non-zero exit = not saved** — report the stderr to the lawyer and stop; never fall back to direct openpyxl writes. One permitted direct write: if a target column's header doesn't exist yet (Related Matters W, Clio Synced X), create the header cell via openpyxl first — header row only — then write the value through the guard.

## Conventions

- **"the lawyer"** in timeline entries refers to the user. Always use "the lawyer" as shorthand for the user in timeline entries — e.g., "the lawyer sent demand letter to opposing counsel."
- **"Client"** refers to the person/entity who retained the lawyer on the matter.

## Spreadsheet Schema

Two sheets — **"Open Matters"** (active files) and **"Closed Matters"** (archived on close) — share one column schema, A–X. Key columns by letter: A File # (immutable `YYYY-NNN`), I Next Action / Deadline, J Timeline, R Limitation Deadline (with P Discovery Date and Q Limitation Statute), S Court Deadlines (JSON array), V Matter Type (soft enum — check existing values on both sheets before writing a variant), W Related Matters, X Clio Synced (create W/X headers on first need).

→ Full schema table (all columns A–X with formats, notes, and per-column rules): REFERENCE.md § Spreadsheet Schema. Read it before creating a tracker, repairing headers, or populating a column whose contract you haven't confirmed this session.

### Formatting & Row Formatting

→ Sheet formatting (header row, font, column widths, wrap columns, data validation) and the row-formatting contract for rows built outside the guard: REFERENCE.md § Sheet & Row Formatting. Read it before Template Creation or any formatting repair the lawyer explicitly requests.

### Write-Time Validation

Date, Next Action, and Court Deadlines formats are enforced by tracker_write.py (exit 2, nothing saved, on violation); full contract in REFERENCE.md § Write-Time Validation Rules.

### Column Ownership

All writes flow through `tracker_write.py`, whichever skill initiates them — other skills read, they don't write. → Primary-writer table and per-column format contracts: REFERENCE.md § Column Ownership.

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

→ Worked examples: REFERENCE.md § Next Action Examples.

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
- If closing, the `close` subcommand appends the closing entry (`YYYY-MM-DD: Matter closed`) automatically — don't add one manually
- On update, merge new entries into the existing timeline in chronological order — never duplicate or delete existing entries
- **Long timelines**: Never compress or summarize timeline entries — keep every entry at full granularity regardless of length. If the cell gets unwieldy in Excel, that's acceptable; the complete record is more valuable than a tidy spreadsheet.

→ Worked example: REFERENCE.md § Timeline Example.

## Audit Columns (K–L)

Columns K and L are simple compliance checkmarks set on every new matter:

- **K — Client ID Verified**: Default to "Pending" on file opening. ID verification is performed via your ID-verification provider (see ID_VERIFICATION_SERVICE in CLAUDE.md; the lawyer sends the client the verification link manually by email). Set to "✓" only once the client has completed verification and the lawyer confirms it. The checkmark should reflect a verification that actually happened (per your law society's client identification and verification rules), not a file-open default. If verification is still outstanding when the file is opened, "Pending" is the accurate state.
- **L — Conflict Check Done**: Set to "✓" on file opening. Confirms a conflicts check was completed before the file was opened. Default to "✓" for all new matters.

Column L (Conflict Check Done) is set to "✓" automatically when a new matter is created, since the conflicts check is run as part of opening the file, and is carried over on close. Column K (Client ID Verified) defaults to "Pending" and flips to "✓" only when verification is actually confirmed, so it is not auto-checked. Neither column requires Gmail searches.

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
- **Q — Limitation Statute**: One of the statute keys defined in `LIMITATION_STATUTES` in your CLAUDE.md (configured for your jurisdiction), or `custom`. If none of the configured statutes apply, use `custom` and ask the user to provide the deadline manually.

  **Presets carry traps — `insurance_act` and `construction_act` mis- or under-capture the real deadlines. See REFERENCE.md § Limitation Statute Preset Caveats before relying on either; confirm with the user and surface the caveat in the confirmation message.**
- **R — Limitation Deadline**: Auto-calculated from P + Q. If the statute is `custom`, the user enters the deadline manually.

When creating a new matter that involves a claim (e.g. Small Claims, demand letter, employment dispute), **always ask the user about the discovery date and applicable limitation** if not obvious from the emails. Flag any limitation period that is within 6 months of expiry.

## Court Deadlines Column (S)

Column S stores court-ordered deadlines as a JSON array. Each entry has three fields:

```json
[
  {"date": "2026-03-25", "description": "Amend claim to add corporation", "source": "March 12 endorsement"},
  {"date": "2026-04-27", "description": "Serve Form 1B on defendants", "source": "scheduling order — 30 days before trial"}
]
```

**Only enter bespoke deadlines** from endorsements, orders, or case-specific requirements — NOT routine rule-based deadlines that the lawyer already knows (e.g. "defence due in 20 days", "disclosure 14 days before settlement conference"). The purpose of this column is to capture the one-off deadlines that come out of specific judicial endorsements and could be missed.

When updating a matter, if a Gmail search or folder scan reveals a new court date, endorsement, or order with a deadline, add it via `court-deadline add` and alert the user. All column S mutations go through the `court-deadline` subcommands, which keep the JSON structure and sort order intact.

**Anchored-relative deadlines.** A deadline defined relative to an unset anchor (e.g. "30 days before trial" with no trial date yet) must NOT be stored with a placeholder date string. Store it with NO "date" field:

```json
{"anchor": "trial date", "offset_days": -30, "description": "Serve Form 1B on defendants", "source": "scheduling order"}
```

Anchored entries are created with `court-deadline add --anchor "..." --offset-days N`. When the anchor date is later confirmed, compute the real date per your jurisdiction's deadline rules (count the period, then roll weekends and court holidays as your rules require), convert the entry with `court-deadline resolve --file-no N --index I --date YYYY-MM-DD`, and push it to the calendar. Overdue-triage surfaces unresolved anchored entries on each sweep.

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

## Related Matters Column

The "Related Matters" column (W) stores comma-separated File #s of sibling matters (create the header at column W on BOTH sheets — Open Matters AND Closed Matters — if absent; header cells via openpyxl are the one permitted direct write; the values then go through the guard's `update`. A header missing on one sheet strands the value when a matter is closed or reopened). When opening a matter for a client or dispute that already has open matters, populate the column both ways — on the new row AND each existing sibling row — and mirror it in a `## Related Matters` section of the brief with relative links to the sibling briefs.

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
9. **Court and scheduling emails are highest priority.** When any email originates from a court address (any domain in `COURT_EMAIL_DOMAINS` in CLAUDE.md, or any court clerk) or references a court file number, scheduling, or hearing date, **always read that message in full** and extract all dates, times, locations, and Zoom/video links. These must be captured verbatim in the Next Action field (with exact date and time) and in the Timeline. Never summarize or skip a court scheduling email.
10. Build the base timeline from Gmail results.
11. **Rate limits and Gmail errors.** If any Gmail call returns a rate-limit, throttle, or 429 error, stop calling Gmail for the rest of this operation immediately — do not retry in a loop and do not keep paginating. Build the timeline from the threads already read plus the folder scan, and tell the user the Gmail pull was cut short by rate limiting and may be incomplete (same disclosure as the "Gmail unavailable" rule). Never present a rate-limited partial pull as a complete record.

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
12. Any hit from either source: surface what was found and where, and wait for the user's direction before opening the file. All three sources clear → proceed normally. If Gmail is unavailable, the third source cannot be cleared: disclose the gap ("conflict check ran on tracker + folders only — Gmail unavailable") and get the lawyer's explicit go-ahead before opening the file.

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
6. **Present to user for confirmation** — one message listing: matter description, opposing party (or "N/A"), Other Parties (or "None identified"), Next Action, contact (email | phone | address, or "not found" for each), and the full draft timeline, ending "Add to tracker? Any corrections?". → Sample message: REFERENCE.md § Sample Confirmation & Review Messages.
7. After confirmation:
   - Locate the tracker (see "Finding the Tracker"). If no tracker exists -> create a new tracker from template (REFERENCE.md § Template Creation) in the Open Files directory first.
   - Run the Matter Folder search (bullet below) so `--matter-folder` can be passed, then create the row:
     `tracker_write.py new-matter --tracker "<tracker>" --client "<Client Name>" --description "<Matter Description>" [--opposing "..."] [--email "..."] [--phone "..."] [--matter-type "..."] [--matter-folder "..."] --date-opened <earliest timeline date, or omit for today>`
     The guard assigns the next File # (max NNN across BOTH Open and Closed Matters sheets, + 1 — never reuses a closed matter's number) and sets Status = "Open", Conflict Check Done = "✓", Client ID Verified = "Pending".
   - Append the timeline: one `timeline --file-no <new File #> --date <event date> --text "..."` call per entry, oldest first. Each call bumps Last Activity per the max rule.
   - Fill the rest in one `update --file-no <new File #>` call (multiple `--set`): Client Address, Other Parties / Related Persons (all non-client, non-opposing parties identified), Next Action / Deadline, `Last Activity=<today>`; add `--set "Client ID Verified=✓"` only if the lawyer confirms Veriff is already complete for this client.
   - **If the matter involves a claim**: ask about discovery date and limitation statute; include `--set "Discovery Date=..."`, `--set "Limitation Statute=..."`, `--set "Limitation Deadline=..."`. Flag if limitation is within 6 months.
   - **If the matter is transactional/advisory**: leave columns P-R unset.
   - Leave Court Deadlines (S) blank unless folder files or emails reveal a specific court-ordered deadline — add each via `court-deadline add`.
   - **Matter Folder (T)**: Search the workspace directory for a subfolder matching the client. **All matching must be case-insensitive.** First, dump the full directory listing to a text file using `ls -1 > /tmp/dirlist.txt`, then grep against that file — this avoids shell issues with special characters (colons, ampersands, parentheses, etc.) in folder names. Try matching against ALL of these permutations of the client name: "First Last" (e.g. "wayne evans"), "Last, First" (e.g. "Taylor, Wayne"), "Last First" (no comma), just the last name, just the first name, and any company/entity name from the matter description or opposing party field. **Also search for the mother/father/third-party name if the matter is brought on someone else's behalf** (e.g. for "Reed v. Blake" brought by Patricia Moore on behalf of June Reed, search for "Moore", "Patricia", "Reed", and "June"). Folders are often named in lowercase or informal formats (e.g. "wayne evans" not "Taylor, Wayne"), or after the entity rather than the person (e.g. "Summit Industries" not "Heaps, Toby"), and frequently contain special characters like colons (e.g. "Patricia Moore : Reed"). Cast a wide net — grep each search term separately and case-insensitively against the text file listing. If found, pass just the subfolder name exactly as it appears on disk via `--matter-folder`. If not found, omit the flag.
7.5. **Create the matter brief.** Write `_matter-brief.md` in the matter folder (column T) — or `_matter-brief-<client-slug>.md` beside the tracker if no folder exists yet — per REFERENCE.md § Matter Brief Template, seeded from the research pull: status bar, Matter Summary, Roles, Open Items, and a `## Tracked Threads` block listing any Gmail threads found. A matter opened without a brief leaves work-on-matter's next session bootstrapping blind.
8. Each guard call has already backed up, saved atomically, and run validate_tracker.py. A non-zero exit means nothing was written — surface the stderr to the lawyer and stop; never retry with direct openpyxl.
9. **Calendar sync**: Invoke the `calendar-sync` skill's `reconcile(new_row)` for this matter. This pushes any limitation date (column R), court deadlines (column S), and dated Next Action (column I) to the Key Dates calendar with the appropriate reminder schedules. Report back to the user: "Pushed N events to Key Dates." If calendar-sync is unavailable, skip this step and note it once — do not block the tracker write.

### 2. UPDATE MATTER

**Trigger**: "update matter [name]" or "update matter"

**Steps**:

1. Extract the client name. If absent, ask.
2. Load the tracker (see "Finding the Tracker"). Find matching row (case-insensitive partial match; if ambiguous, ask).
3. **Run the Core Research Procedure** (folder scan + Gmail from Last Activity date onward).
3.5. **Conflict re-check on newly discovered parties.** Every new party the pull surfaces (co-defendants, guarantors, principals behind entities, new opposing counsel) must be checked against columns B, C, H, and U on BOTH sheets (case-insensitive). Match as client (col B): stop and alert the lawyer of a potential conflict before proceeding. Match as opposing party (col H): note as repeat-litigant intel and continue. Any match in column U: surface for review. Then append the new parties to column U.
3.6. **Limitation capture on claim emergence.** If Matter Type (col V) is Advisory, Transactional, or blank but the pull surfaces evidence of a claim (a filed court document, a formal demand received or sent, or a document explicitly asserting a cause of action), ask the lawyer for the discovery date and limitation statute. NEVER auto-populate P/Q/R from inferred evidence — only the lawyer can confirm the limitation analysis. Populate per the Limitation Period Columns section. Flag if the deadline is within 6 months.
4. Read the existing Timeline from the spreadsheet.
5. Merge new events into the existing timeline in chronological order. Do not duplicate or remove existing entries.
6. **Present the updated timeline to user for confirmation** — existing entries plus [NEW]-tagged additions, the updated Next Action, and whether to also update the matter description (quote the current one); ask to confirm. → Sample message: REFERENCE.md § Sample Confirmation & Review Messages.
7. After confirmation, write the changes through the guard:
   - New timeline events: one `timeline --file-no N --date <event date> --text "..."` call per entry, in chronological order (each appends to column J and bumps Last Activity). If a new event predates existing entries and its position in the cell matters, write the full merged timeline instead via `update --set "Timeline=<merged text>"` — never duplicate or drop existing entries.
   - Everything else in one `update --file-no N` call (multiple `--set`): Next Action / Deadline; Matter Description if scope changed; Client Email/Phone/Address if new contact info found; Other Parties / Related Persons if new parties were identified; `Last Activity=<today>`.
   - If folder files or emails reveal a new court date, endorsement, or order with a deadline, add it via `court-deadline add` and alert the user
   - **Expired court deadlines — confirm before removing.** A passed date does not mean the obligation was satisfied; it may have been MISSED, which is exactly what most needs surfacing. For each column S entry whose date has passed, check the research pull for evidence it was satisfied (filing confirmation, email, endorsement). Evidence found: remove it with `court-deadline remove --index I` and tell the user ("Cleared: 2026-02-15 — Amend claim — amended claim filed Feb 14"). No evidence: do NOT remove — flag it ("2026-02-15 — Amend claim: deadline passed, no evidence it was done. Was this handled?") and wait for the answer. Adjourned or superseded with a replacement entry already in column S (or being added now): remove the stale entry and record the adjournment in the timeline. Silent removal would also gut the overdue-triage skill, whose job is to investigate exactly these entries.
   - If the matter now involves a claim but limitation columns (P-R) are blank, flag this and ask the user about discovery date and statute
   - **If column T (Matter Folder) is blank**, attempt to populate it now using the folder resolution logic from the NEW MATTER workflow (`update --set "Matter Folder=..."`).
8. Each guard call has already backed up, saved atomically, and run validate_tracker.py. A non-zero exit means nothing was written — surface the stderr to the lawyer and stop; never retry with direct openpyxl.
9. **Write/refresh `_matter-brief.md`** in the matter folder (column T). If the brief exists, read it and update with current information. If it doesn't exist, create it. Preserve `## Tracked Threads` (append new threads, advance last-seen dates — never rewrite) and `## Resolved / Historical` (append-only) verbatim. MERGE the live sections (Roles, Risks, Positions, Open Items) rather than appending duplicates; demote resolved items to `## Resolved / Historical`; never silently prune content. The brief is a current-state snapshot of the matter (soft warning at 250 lines, no hard cap — see "Matter Brief Format" below). This step happens automatically after saving the tracker — no need to ask the user for separate confirmation.
10. **Calendar sync**: Invoke `calendar-sync.reconcile(updated_row)`. This creates new events for any newly-added deadlines, updates events whose dates or descriptions changed, and deletes events for deadlines that were pruned (e.g., expired court deadlines removed from column S). Report the diff to the user: "Calendar sync: X added, Y updated, Z removed." If any events were deleted because a deadline passed, name them explicitly so the user sees what's no longer on the calendar.

### 3. CLOSE MATTER

**Trigger**: "close matter [name]" or "close matter"

**Steps**:

1. Extract the client name. If absent, ask.
2. Load the tracker (see "Finding the Tracker"). Find matching row on the "Open Matters" sheet.
3. **Run the Core Research Procedure** (folder scan + Gmail from Last Activity date onward) — capture any final correspondence.
4. Merge any new events into the existing timeline.
5. Show the closing entry (`YYYY-MM-DD: Matter closed`) as the final timeline entry in the draft — the `close` subcommand appends it at write time; do not append it manually.
6. **Populate any blank columns before closing:**
   - If column T (Matter Folder) is blank, attempt to populate it using the folder resolution logic (the folder scan in step 3 already identified the path — write the subfolder name).
   - If columns M-O are blank but contact info was found during the research pull, populate them now.
   - If column U is blank but other parties were identified, populate it now.
7. **Present to user for confirmation** — client and matter description, the full merged timeline including the closing entry, and a note that this will move the matter to the "Closed Matters" tab; ask "Confirm close?". → Sample message: REFERENCE.md § Sample Confirmation & Review Messages.
8. After confirmation:
   - Merge any new timeline events via `timeline` calls (one per entry).
   - One `update --file-no N` call: `--set "Next Action / Deadline=FILE CLOSED"` plus any blank columns filled in step 6 (Matter Folder, Client Email/Phone/Address, Other Parties / Related Persons).
   - `close --file-no N` — moves the row to "Closed Matters", sets Status = "Closed" and Date Closed = today, appends the closing timeline entry, and bumps Last Activity. Never move the row with openpyxl. If the guard reports the "Closed Matters" sheet is missing, create that sheet per REFERENCE.md § Template Creation (header row only), then re-run.
9. Each guard call has already backed up, saved atomically, and run validate_tracker.py. A non-zero exit means nothing was written — surface the stderr to the lawyer and stop; never retry with direct openpyxl.
10. **Update `_matter-brief.md`** in the matter folder — append "FILE CLOSED" to the summary and mark open items as resolved or moot. If no brief exists, create a final one for the closed file.
11. **Calendar sync**: Invoke `calendar-sync.cancel_all_for_matter(file_number)` to remove every event on Key Dates for this file — court, limitation, follow-ups, and any third-party prompts. Confirm: "Cancelled N events on Key Dates." Closed files should leave no trace on the calendar.

### 4. REVIEW OPEN MATTERS

**Trigger**: "show my open files", "what's open", "matter list", "file summary"

**Steps**:

1. Load tracker.
2. Read the "Open Matters" sheet.
3. Display a clean summary in conversation — numbered lines: File # | Client | Matter | Next action | Last activity. → Sample display: REFERENCE.md § Sample Confirmation & Review Messages.

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
3. Display a clean summary in conversation — numbered lines: File # | Client | Matter | Opened | Closed. → Sample display: REFERENCE.md § Sample Confirmation & Review Messages.

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
3. **Present to user for confirmation** — File #, client, matter description, Date Closed, and a note that this moves the matter back to "Open Matters" and sets Status to Open; ask "Confirm reopen?". → Sample message: REFERENCE.md § Sample Confirmation & Review Messages.
4. After confirmation:
   - `reopen --file-no N` — moves the row back to "Open Matters", sets Status = "Open", clears Date Closed, appends the reopen timeline entry (`YYYY-MM-DD: Matter reopened`), and sets Last Activity to today. Never move the row with openpyxl.
   - Ask the user what the next step is (the previous "FILE CLOSED" entry is no longer valid), then `update --file-no N --set "Next Action / Deadline=..."`.
5. A non-zero exit from either guard call means nothing was saved — report the stderr to the lawyer and stop.
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

A brand-new tracker file (or a missing "Closed Matters" sheet) is built with openpyxl. → Full build spec (sheets, header formatting, data-validation lists, tab colour, page setup): REFERENCE.md § Template Creation. Read it before creating a tracker file or sheet from scratch.

## File Number Assignment

Assigned by `tracker_write.py new-matter` — never hand-computed:

- Format: `YYYY-NNN` where YYYY is the current year and NNN is a zero-padded sequential number
- If the tracker is new, numbering starts at `{current_year}-001`
- The guard scans **all** File # values across both Open and Closed Matters sheets to find the highest number for the current year, then increments by 1. This prevents collisions when files have been closed and removed from Open Matters.
- If the year has changed since the last entry, numbering resets to `{new_year}-001`

## Important Behaviour Rules

1. **Every command does a full research pull (Gmail + folder scan).** New, update, and close all search Gmail and scan the client folder to build/refresh the timeline. No exceptions.
2. **Always confirm before writing.** Never add, update, or close a matter without showing the user the proposed timeline and getting explicit approval.
3. **Folder files and Gmail are supplementary, not authoritative.** The timeline is drafted from local files and emails but the user's corrections override everything.
4. **Timelines are append-only on update.** When merging, never delete or alter existing timeline entries. Only add new ones in chronological position.
5. **Find the tracker automatically.** See "Finding the Tracker" section below. If no tracker exists, create a fresh one in the Open Files directory.
6. **Preserve existing data.** When editing the tracker, never overwrite or delete existing rows. Only append or modify the targeted row.
7. **Flag if Gmail is unavailable or rate-limited.** If Gmail MCP tools are not available in the current environment, build the timeline from folder contents and tell the user. If a Gmail call returns a rate-limit / throttle / 429 error, stop calling Gmail for the rest of the operation, build from whatever was already pulled plus the folder scan, and disclose that the email pull was incomplete. If both Gmail and folder are empty, proceed with manual entry — ask them to dictate the timeline events.
8. **Every write goes through tracker_write.py.** The guard makes a timestamped backup into `backups/` beside the tracker, saves atomically, and runs validate_tracker.py on every write. A non-zero exit means nothing was saved — surface the stderr to the lawyer and stop; never retry with direct openpyxl. On a validation FAIL (exit 3), point the lawyer to the backup path the guard prints. Do NOT delete older backups -- let the folder accumulate history. The user can prune manually if it ever grows too large.
9. **Excel lock files.** The guard refuses to write while a lock file (`~$matter-tracker.xlsx`) exists. If a call fails on the lock, tell the user: "The tracker appears to be open in Excel. Close it and I'll re-run." Use `--force-unlocked` only when the lawyer confirms the lock is stale.
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
5. **If no tracker exists at all**, create a fresh one from the template (see REFERENCE.md § Template Creation) in the directory the user specifies.

Once found, remember the tracker path for the rest of the session.

## Locating the Matter Folder for Briefs

When writing `_matter-brief.md` after an update or close, you need to resolve the Matter Folder name from column T to an actual path on disk. The tracker and matter folders live in the same parent directory (the Open Files directory).

**Primary approach**: The client folder is a sibling directory of the tracker file. List the subdirectories in the same directory as `matter-tracker.xlsx` and find the one matching column T. **Then resolve the matter-specific subfolder, exactly as work-on-matter Step 2 does**: list the client folder's immediate contents; if a subfolder matches the matter description (column C) or opposing party (column H) by keyword, that subfolder is the matter folder and the brief belongs there; if no subfolder matches but `_matter-brief.md` already exists at the client folder's top level, use the top level; if subfolders exist, none match, and there is no top-level brief, ask the user which subfolder. Write the brief to the resolved folder. The reason this matters: writing at the client folder's top level when the work lives in a subfolder is how a multi-matter client ends up with two divergent briefs — or has matter A's brief overwritten with matter B's content.

**Fallback — column T is blank**: List all sibling directories and do a case-insensitive fuzzy match against the client name (try last name, first name, entity name, and permutations). If a match is found, use it and populate column T for future use. If no match, ask the user for the folder path.

**Fallback — no match found**: Save the brief to the same directory as the tracker as `_matter-brief-[client-name].md` and tell the user to move it to the matter folder. Don't let the brief step fail silently — the user should know where it ended up.

## Matter Brief Format

When writing or refreshing the matter files (triggered by new-matter, update, or close operations), use the three-file architecture defined in the work-on-matter skill: `_matter-brief.md` (snapshot — live sections replaceable, no hard cap; superseded items demote to `## Resolved / Historical`), `_matter-decisions.md` (append-only log of strategic reasoning, no cap), and `_matter-comms.md` (append-only log of file-specific operational rules, no cap). All three live in the matter folder.

**This skill's update workflow primarily writes the brief.** Update and close operations rebuild the brief from a fresh research pull. The decisions log and comms preferences are owned by the work-on-matter skill — this skill should never overwrite, edit, or prune them. If those files exist in the matter folder, leave them alone. (The exception: if the close operation surfaces a final closeout note that future sessions should be able to find, append it to the decisions log as a new entry, never edit existing entries.)

**Brief length: soft warning at 250 lines, no hard cap.** If a save would push the brief past 250 lines, surface it to the user before writing: "Brief is at [N] lines. Want me to refactor (demote superseded Risks/Positions/Open Items to `## Resolved / Historical`, move reasoning to _matter-decisions.md) or save as-is?" Wait for direction. Do NOT auto-refactor — the model can't reliably tell which items are still live versus superseded, and a silent change can drop content that should have been kept. Demote, never delete: same semantics as the work-on-matter skill.

If the content pushing the brief long is reasoning rather than current state, propose moving it to `_matter-decisions.md` as part of the refactor offer.

This 250-line soft warning matches the work-on-matter skill so briefs produced by either skill are sized consistently.

**Brief format (`_matter-brief.md`):** mandatory section order — privilege header, `# [Client Name] — [File #]` heading, 3-line status bar (ACTIVE DEADLINE / LAST ACTION / AWAITING), Matter Summary, Current Stage, Roles, Risks & Issues Flagged, Positions Taken / Advice Given, Open Items, Key Terms / Provisions, Tracked Threads, Resolved / Historical, Last Updated. → Full template with placeholder text: REFERENCE.md § Matter Brief Template. Read it before creating or rebuilding a brief.

Omit any section that doesn't apply (e.g., skip "Key Terms" for a litigation matter), except the Roles block, which is mandatory for every brief.

**Backup before every brief write.** Copy the existing `_matter-brief.md` (if any) to `backups/_matter-brief.YYYY-MM-DD.md` in a `backups/` subfolder of the matter folder. One backup per day; same-day overwrites fine. Verify the file re-opens cleanly after write; on failure, point the user to the most recent backup.

**Decisions log and comms preferences format and lifecycle:** Defined in the work-on-matter SKILL.md ("Decisions Log Format" and "Comms / Client Preferences Format" sections). This skill does not create or edit those files except as noted above for closeout notes.

**The Roles block is mandatory.** Every named party in the matter with their role and a source citation for that role. This is the single place in the brief where each person is pinned to a source. Paraphrasing a role in an outgoing email without first confirming it here is how role errors leak into client-facing work. → Example Roles block: REFERENCE.md § Matter Brief Template.

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

## Client Profile for Repeat Clients

Repeat clients get ONE `_client-profile.md` in the client's top-level folder: contact preferences, communication style, sensitivities, billing quirks. Matter briefs reference it ("See _client-profile.md") instead of re-documenting. When opening a second matter for an existing client, create the profile from the first matter's notes if it doesn't exist.
