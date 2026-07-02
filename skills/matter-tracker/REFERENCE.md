# Matter Tracker — Reference

Reference for the matter-tracker skill — read the section the SKILL.md step points you to.

## Spreadsheet Schema

The tracker spreadsheet uses two sheets: **"Open Matters"** (active files) and **"Closed Matters"** (archived files). Both sheets share the same column schema:

### Sheet 1: "Open Matters" — all active files
### Sheet 2: "Closed Matters" — archived files moved here on close

Both sheets use these columns:

| Column | Header | Format | Notes |
|--------|--------|--------|-------|
| A | File # | Text (e.g. "2026-001") | Auto-assigned: YYYY-NNN. Next number = highest NNN across **both** the Open Matters AND Closed Matters sheets, + 1. A closed matter keeps its number forever — never reuse it. (Scanning only Open Matters caused duplicate 2026-231: a matter closed the same day freed its number on the Open sheet and a new matter took it.) |
| B | Client Name | Text | Primary client name. **Standard format: `Entity Name (Principal Name)`** — e.g. "Acme Group Inc. (Blake Murphy)". For individual clients with no entity, just use their name. For individuals acting through a numbered company, lead with the entity: "10014056 Holdings LLC (Jane D)". If multiple key individuals exist (e.g. two directors), comma-separate them inside the brackets: "ABC Real Estate Solutions Inc. (Bob Adams, Carol Chen)". **Never use slash format** (e.g. "Name / Corp") — always use the brackets format for consistency. This ensures the conflict check catches both the entity and the individual(s) behind it. |
| C | Matter Description | Text (wrap text) | Brief description of the engagement |
| D | Status | Text | "Open" or "Closed" |
| E | Date Opened | Date (YYYY-MM-DD) | Date the file was opened |
| F | Date Closed | Date (YYYY-MM-DD) | Blank until closed |
| G | Last Activity | Date (YYYY-MM-DD) | Updated on every interaction |
| H | Opposing Party | Text | If applicable; blank otherwise |
| I | Next Action / Deadline | Text (wrap text) | Key upcoming deadline or next step — see SKILL.md § Next Action Format |
| J | Timeline | Text (wrap text) | Concise chronological timeline built from Gmail and client folder files — see SKILL.md § Timeline Format |
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
| V | Matter Type | Text | Free-text classification of the matter (e.g. "Litigation", "Solicitor", "Transactional", "Advisory", "Small Claims", "Demand Letter"). No fixed enum — keep values consistent with prior rows for filterability, but allow new categories as the practice evolves. Used for sorting and filtering matters in reports; not client-facing. **Soft enum**: before writing a value, list the existing unique values from both sheets; if the new value is a near-duplicate of an existing one (e.g. "Small Claims" vs "SCC" vs "Litigation - Small Claims"), propose the existing value and require explicit confirmation to create a variant. Reject placeholder values like "Test". |
| W | Related Matters | Text | Comma-separated File #s of sibling matters for the same client or dispute (see SKILL.md § Related Matters Column). Create the header at column W on first need. |

## Sheet & Row Formatting

### Formatting

- **Header row**: Bold, light blue fill (#D6E4F0), frozen pane, auto-filter enabled
- **Font**: Arial 10pt throughout
- **Column widths**: A=12, B=22, C=40, D=10, E=14, F=14, G=14, H=22, I=30, J=60, K=14, L=14, M=22, N=16, O=30, P=14, Q=30, R=14, S=40, T=50, U=50, V=18
- **Status column**: Use data validation (Open/Closed)
- **Limitation Statute column (Q)**: Use data validation — list = "limitations_act_basic,limitations_act_ultimate,cpa_2_year,employment_standards,human_rights,construction_act,insurance_act,municipal_liability,custom"
- **Wrap text** on columns C, I, J, S, and U **only** — do NOT set wrap_text on other columns

### Row Formatting (CRITICAL — must match existing rows exactly)

Row values are written by `tracker_write.py`, which preserves existing cell formatting on save. This section defines the formatting contract — it applies directly only when building rows outside the guard (Template Creation, or a formatting repair the lawyer explicitly requests). When building such a row, **clone the formatting from the nearest existing data row** to ensure visual consistency. Specifically:

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

## Write-Time Validation Rules (enforced by tracker_write.py)

The write guard enforces these rules on every write — hard violations are rejected (exit 2, nothing saved); soft violations produce a warning but still write. The skill no longer self-polices formats at write time. The rules, for reference:

1. **Date columns E, F, G, P, R**: `YYYY-MM-DD` only — never a datetime ("2025-09-01 00:00:00" has appeared in the wild) and never prose (a Last Activity cell was once found holding action text). Prose belongs in Next Action or the Timeline.
2. **Next Action (I)**: ONE line — the guard warns above 80 chars and hard-rejects above 200 or any multi-line value. Leading "YYYY-MM-DD: " ONLY when that date is the actual trigger. Multi-item blocker lists go in the brief's Open Items, not the tracker.
3. **Court Deadlines (S)**: every "date" field must be a real YYYY-MM-DD date. Placeholder values like "TBD-30" are rejected — use the anchored-entry format in SKILL.md § Court Deadlines Column (S) instead.

## Column Ownership

All writes flow through `tracker_write.py`, whichever skill initiates them. Primary writer(s) and format contract per written column — other skills read, they don't write:

| Col | Primary writer(s) | Contract |
|-----|-------------------|----------|
| A File # | matter-tracker | Immutable once assigned |
| G Last Activity | matter-tracker, work-on-matter, overdue-triage | Date-only; set to max(current, event date); never a future date |
| I Next Action | matter-tracker, work-on-matter, daily-triage (with the lawyer approval) | Single line per SKILL.md § Next Action Format |
| J Timeline | matter-tracker, work-on-matter | Append-only; event-dated entries |
| M / U | daily-triage (auto-fill when blank), matter-tracker | Per column definitions |
| P/Q/R Limitation | matter-tracker (with the lawyer's approval) | R never cleared without claim-filed confirmation |
| S Court Deadlines | matter-tracker, overdue-triage | JSON contract per SKILL.md § Court Deadlines Column (S), incl. anchored entries |

## Next Action Examples

Examples:
```
2026-03-27: Settlement conference at 1:15 PM
2026-03-19: 7-day cure period expires; file for judgment
Send draft claim to insurer with response deadline
Awaiting client instructions re: citizenship oath
```

## Timeline Example

Example:
```
2026-01-15: Initial client intake call re: share purchase
2026-01-18: Sent engagement letter to client
2026-01-22: Received draft SPA from opposing counsel (Davies LLP)
2026-02-01: Sent markup of SPA to opposing counsel
2026-02-10: Client approved final SPA
2026-02-14: Closing — executed SPA exchanged
```

## Limitation Statute Preset Caveats

**Presets can carry traps — confirm with the user before relying on them, and surface the caveat in the confirmation message:**
- Some statutes have short special periods that apply only to narrow claim types. Never default to the shorter period just because the subject matter superficially fits — ask which period applies.
- Some statutes generate procedural deadlines a simple period preset does NOT capture (e.g. lien preservation and perfection windows in construction matters). Track those explicitly in column S or column I so they reach the calendar.

## Sample Confirmation & Review Messages

Templates for the confirmation prompts and review displays. The field lists are defined in the SKILL.md workflow steps; substitute real values.

### NEW MATTER — confirmation (workflow step 6)

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

### UPDATE MATTER — confirmation (workflow step 6)

   ```
   Updated timeline for [Name]:
   [existing entries]
   [NEW] YYYY-MM-DD: [new event]
   [NEW] YYYY-MM-DD: [new event]

   Next Action: [updated next step/deadline]
   Also update matter description? Currently: "[current]"
   Confirm?
   ```

### CLOSE MATTER — confirmation (workflow step 7)

   ```
   Closing file for [Name] — [Matter Description]
   Final timeline:
   [full merged timeline including CLOSED entry]

   This will move the matter to the "Closed Matters" tab.
   Confirm close?
   ```

### REOPEN MATTER — confirmation (workflow step 3)

   ```
   Reopening: File #[X] | [Client Name] — [Matter Description]
   Closed on: [Date Closed]

   This will move the matter back to "Open Matters" and set Status to Open.
   Confirm reopen?
   ```

### REVIEW OPEN MATTERS — display (workflow step 3)

```
You have X open matters:

1. 2026-001 | Smith, J. | Share purchase agreement | Next: Awaiting signed docs | Last: 2026-03-01
2. 2026-002 | Patel, R. | Damage deposit — Small Claims | Next: 2026-04-15 Settlement conf. | Last: 2026-03-10
...
```

### REVIEW CLOSED MATTERS — display (workflow step 3)

```
You have X closed matters:

1. 2025-003 | Lee, D. | Lease dispute — Small Claims | Opened: 2025-09-01 | Closed: 2026-01-15
...
```

## Matter Brief Template

```markdown
> PRIVILEGED & CONFIDENTIAL — Solicitor-Client Privilege / Work Product

# [Client Name] — [File #]

ACTIVE DEADLINE: [next dated deadline, or "none"]
LAST ACTION: [YYYY-MM-DD — one line]
AWAITING: [who owes what]

## Matter Summary
[2-3 sentences: what this matter is about, who the parties are, what stage it's at]

## Current Stage
[Where in the process; next critical date]

## Roles [last update: YYYY-MM-DD]
- Name (role) — source: [email date / doc filename + page / tracker col X]
- Name (role) — source: [...]

## Risks & Issues Flagged [last update: YYYY-MM-DD]
- [Concise bullet points of flagged risks, unusual provisions, practical concerns]

## Positions Taken / Advice Given [last update: YYYY-MM-DD]
- [Key advice given, positions taken in negotiations, strategic decisions made — current state, not historical reasoning. Reasoning lives in _matter-decisions.md.]

## Open Items [last update: YYYY-MM-DD]
- [What's still unresolved, pending, or needs follow-up]

## Key Terms / Provisions
[Only for transactional matters — price, term, material conditions, unusual clauses]

## Tracked Threads
- [Gmail thread ID] — "[short subject label]" — last seen YYYY-MM-DD

## Resolved / Historical
- [YYYY-MM-DD — Demoted item with one-line resolution note. Append-only.]

## Last Updated
[Date of this update]
```

**Example Roles block:**

```
## Roles
- Ben Mercer (Landlord principal, Metro Fashions Ltd.) — source: Lease Assignment executed Apr 7 2026, recital A
- Frank West (counsel for Assignor / Seller, 1234567 Ontario Inc.) — source: signature block of Lease Assignment; confirmed Apr 2 14:04 ET email
- Rita Moss (Metro Controller) — source: Apr 17 11:39 email re security deposit wire
- KRB Lawyers Inc. (counsel of record for Landlord) — source: s.2.8.1(c) of Lease Assignment
```

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
