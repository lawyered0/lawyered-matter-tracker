---
name: overdue-triage
description: "Use this skill for a big periodic sweep of the matter tracker to clean up stale or expired deadlines. Trigger on: 'overdue triage', 'check overdue', 'overdue items', 'overdue cleanup', 'expired deadlines', 'stale deadlines', 'clean up deadlines', 'run the overdue sweep', 'deadline cleanup', or any request to review past-date items across every open matter and reconcile against reality. Scans all open matters for past dates in Next Action (col I), Limitation Deadline (col R), and Court Deadlines (col S), investigates each via Gmail and the matter folder, confirms with the lawyer one item at a time, then applies approved changes in a single batched write. Also produces a red-flag list of unresolved items with suggested next actions. Heavier cousin of daily-triage, meant to run every few weeks, not daily. Do NOT trigger on 'daily triage', 'morning check', or routine email review: those belong to daily-triage."
---

# Overdue Triage: Big Sweep Cleanup

## Purpose

The matter tracker accumulates stale deadlines. When a settlement conference happens, a court deadline passes, or a limitation is resolved by filing, the tracker doesn't auto-update. It relies on the user running `update matter [name]` on that specific file. Over time, columns I, R, and S fill up with past dates that no longer reflect reality.

This skill does one thing well: **sweep every open matter for overdue items, figure out which ones were already dealt with, and clean them up in one confirmed batch of guarded writes.** Items that look unresolved get surfaced as a red-flag list with suggested next actions.

This is the "once in a while" big triage. Daily email triage belongs to `daily-triage`.

## Conventions

- **"the lawyer"** refers to the user.
- **"Client"** refers to the retained party.
- **"Overdue item"** = any dated entry in columns I, R, or S where the date is before today.

## Dependencies

- **Matter tracker spreadsheet**: `matter-tracker.xlsx`, located using the same CWD, parent, grandparent resolution as matter-tracker.
- **Gmail MCP tools**: `search_threads` (find threads by query) and `get_thread` (read a full thread by ID). There is no message-level search and no single-message read — everything is thread-level. `search_threads` truncates the per-thread message list, so use it to discover thread IDs and read each thread in full with `get_thread`. If unavailable, fall back to folder-scan only and warn the user.
- **Local file tools**: Glob, Grep, Read for folder scans.
- **calendar-sync skill**: After the batched tracker write, invoke `calendar-sync.reconcile(row)` for every matter touched, so resolved deadlines disappear from Key Dates.

## Tracker Writes

Tracker writes go through tracker_write.py — never ad-hoc openpyxl (openpyxl is fine for reading):

```
python3 "<tracker dir>/scripts/tracker_write.py" <subcommand> --tracker "<tracker path>" ...
```
 `<tracker dir>` is the directory containing matter-tracker.xlsx — build the path from the tracker's own location (the scripts ship beside it in `scripts/`), so the guard stays reachable in sandboxed sessions where only the client folder is mounted. If the guard is somehow unreachable, PAUSE and flag it to the lawyer — proceed with a manual write (replicating backup + validation) only on the lawyer's explicit go-ahead, never silently.

Subcommands this skill uses: `update --file-no N --set "COLUMN=value" [--set ...]` · `timeline --file-no N --date YYYY-MM-DD --text "..."` (appends and bumps Last Activity per the max rule) · `court-deadline remove --file-no N --index I` · `court-deadline resolve --file-no N --index I --date YYYY-MM-DD`. Every call does the Excel-lock check, its own timestamped backup into `backups/`, an atomic save, and runs validate_tracker.py automatically. **Non-zero exit = not saved** — report the stderr to the lawyer; never fall back to direct openpyxl writes.

## Detection Logic

The three overdue sources are:

### 1. Column I (Next Action / Deadline): past-date Next Action

Parse column I with the regex `^(\d{4}-\d{2}-\d{2})\s*:`. The date MUST be at the start of the string and followed by a colon. This matches the matter-tracker schema convention (`YYYY-MM-DD: [description]`). Do NOT use `\b(\d{4}-\d{2}-\d{2})\b` (searches anywhere in the string): that produces false positives when a historical or contextual date appears in prose (e.g., "adjourned from 2026-04-09", "Form 9B filed 2026-01-21"). If no leading date matches, the Next Action is open-ended and NOT overdue.

- Example overdue: `"2026-03-27: Settlement conference at 1:15 PM"` (leading date, past)
- Example NOT overdue: `"Awaiting client instructions re: oath"` (no leading date)
- Example NOT overdue: `"2026-05-15: Serve disclosure"` (leading date, future)
- Example NOT overdue: `"Settlement conference to be rescheduled (adjourned from 2026-04-09); awaiting new date"` (date is embedded, not leading, so no deadline)
- Example NOT overdue: `"Awaiting court to schedule assessment hearing (Form 9B filed 2026-01-21)"` (embedded filing date, not a deadline)

**Stick to the strict leading-date regex above.** A looser search like `\b(\d{4}-\d{2}-\d{2})\b` (matching a date anywhere in the string) would mis-flag embedded historical dates such as "adjourned from 2026-04-09" as overdue. The leading-date-plus-colon anchor is what prevents those false positives.

### 2. Column R (Limitation Deadline): past limitation expiry

If column R is a date before today, the limitation has expired. This is the highest-stakes category. Never auto-clear without positive evidence the claim was filed or the user affirmatively tells you to. An expired limitation that wasn't filed is a potential malpractice issue; surface it loudly.

### 3. Column S (Court Deadlines): past entries in the JSON array

Parse column S as JSON. For each entry `{"date": "...", "description": "...", "source": "..."}`, if `date` is before today, that deadline is overdue. Entries can be independently resolved. Remove individual expired entries, not the whole array.

**Anchored-relative entries.** Entries with an `anchor` field and no `date` (the representation for "30 days before trial"-type deadlines; legacy `"TBD-*"` date strings get the same treatment) have no concrete date yet. Surface them EVERY sweep as "unresolved relative deadline: anchor not yet set". When the lawyer provides the anchor date, compute the concrete date per your jurisdiction's deadline rules (roll weekends and court holidays as required), convert the entry with `court-deadline resolve --index I --date YYYY-MM-DD` in the batch, and push to calendar via calendar-sync.

## Workflow

### Step 1: Load Tracker and Detect

1. Locate and load `matter-tracker.xlsx` (CWD, parent, grandparent, then ask).
2. Check for lock file `~$matter-tracker.xlsx`. If present, stop and tell the lawyer to close Excel.
3. Read **"Open Matters"** only (closed matters are archived; don't touch them).
4. **Data quality pre-scan.** Before building the overdue list, sweep every open row for column-level data problems. Don't fail the scan. Collect warnings and surface them at the top of the final output. Checks:
   - Column G (Last Activity) non-empty but not a valid `YYYY-MM-DD` date (e.g., prose text written into the wrong cell).
   - Column I (Next Action) contains an embedded date but no leading date (often a sign the user meant to set a deadline and put it in the middle of the sentence).
   - Column R (Limitation Deadline) populated but column Q (Limitation Statute) is blank, or vice versa.
   - Column S (Court Deadlines) is not valid JSON (parse error).
5. For each matter row, extract:
   - File # (A), Client Name (B), Matter Description (C), Last Activity (G), Opposing Party (H), Next Action (I), Limitation Deadline (R), Court Deadlines (S), Other Parties (U), Matter Folder (T)
6. Build the overdue list. Each entry has:
   - `file_no`, `client`, `description`, `last_activity`, `matter_folder`
   - `overdue_type`: one of `"next_action"`, `"limitation"`, `"court_deadline"`
   - `overdue_date`: the past date (YYYY-MM-DD)
   - `overdue_text`: the description (e.g., "Settlement conference at 1:15 PM", "Amend claim to add corporation")
   - For court deadlines: `index` within the JSON array (needed to remove the specific entry)
7. **Cross-reference same-date events (col I + col S).** After building the raw list, look for duplicates: if a matter has a Next Action with leading date D AND a Court Deadline with date D (within 1 day tolerance, same description keywords), group them as a single "event" with both tracker touchpoints. When the lawyer confirms resolution on a grouped event, the write touches both columns in the batch (remove the col S entry AND replace the col I Next Action). This avoids asking the same question twice for what's really one hearing.
8. Sort the overdue list by `file_no`, then by `overdue_date` ascending. Grouped events sort by their shared date.
9. **Report the scan result before investigating:**

   ```
   Overdue scan: N items (G grouped events) across M matters.
     Court deadlines past date:  X
     Next Action dates past:     Y
     Limitation deadlines past:  Z (HIGH RISK, review carefully)
     Grouped (col I + col S same date): G

   Data quality warnings (K):
     * File #XXXX-XXX: Last Activity column contains prose, not a date
     * File #XXXX-XXX: Next Action has embedded date but no leading deadline
     ...

   Starting investigation. Gmail + folder scan per matter, then I'll walk you through each one for confirmation.
   ```

### Step 2: Per-Matter Investigation (chunked)

Group the overdue list by `file_no` (all overdue items on the same matter share the same investigation pull).

**Chunking.** Process matters in batches of **8** (configurable). Do not attempt all 30+ matters in one continuous pull. Gmail thread reads add up fast, and a 20+ minute silent stretch with no feedback is a poor UX. After each batch, emit a progress line:

```
Investigated 8/38 matters (3 with resolved evidence, 2 unresolved, 3 ambiguous). Continuing...
```

After every batch, offer an implicit bail-out: if the lawyer interrupts, the decisions made so far are preserved. Store in-progress evidence in a working JSON at `/tmp/overdue-triage-session.json` (or the OS-appropriate temp dir) keyed by `file_no`. On resume, skip matters already investigated this session.

For each matter in a batch:

1. **Gmail pull (two-pass when sender addresses are known).** A keyword-only pull will silently miss short replies on old threads ("got it, see you Friday" from opposing counsel) because Gmail matches each message individually and a one-line reply carries no matter-specific text. The lookback for overdue triage is often weeks or months back to the earliest overdue date, which makes this failure mode more likely, not less. Run both passes:

   **Pass A — Keyword pass.** Build the query from these inputs, joined with OR:
   - Client name from column B (entity AND principal name in brackets, as separate OR'd terms — strip the parenthetical and search both forms).
   - Opposing party from column H if populated.
   - Named role-holders from `_matter-brief.md` `## Roles` if the brief exists (opposing principals, opposing counsel, witnesses, experts, agents). Skip the client (covered) and skip generic role labels.
   - Matter-specific keywords from column C that are unusual enough to be useful (e.g., property address, court file number).

   Use `newer_than` based on the earliest overdue date for that matter, minus 7 days for context.

   **Pass B — From-address pass.** Build the query as `(from:addr1 OR from:addr2 OR ...) newer_than:Xd` (same window as Pass A), using:
   - Client Email from tracker column M.
   - Email addresses parsed out of `_matter-brief.md` `## Roles` (opposing counsel, opposing principals, third parties).
   - Court / tribunal addresses that have appeared on this matter before (look in earlier timeline entries on column J).

   Pass B is **non-negotiable when any addresses are on file** — without it, opposing-counsel one-liners on old hearing threads silently slip through and you miss the evidence that the deadline was met. Pass B may be skipped only when no addresses are anywhere on file (rare for matters old enough to have overdue items). When skipped, note it explicitly in the per-matter evidence bundle so the lawyer knows the coverage limit.

   Union the thread IDs from both passes and dedupe. Read every unique thread in full with `get_thread`. Snippets lie.
2. **Folder scan** (if column T is populated): `Glob` the matter folder for common legal file types (`**/*.pdf`, `**/*.docx`, `**/*.msg`, `**/*.eml`). Filter to files modified after the earliest overdue date minus 7 days.
3. **Read** the relevant files. For **scanned PDFs** where text extraction returns empty (common for court endorsements, affidavits of service, filed originals produced by the court or a process server): treat the **filename** and **modification date** as primary evidence of occurrence (e.g., `"ENDORSEMENT RECORD - DJ WALLACH - 27 MAR 2026.pdf"` on disk strongly implies the hearing produced an order). Do not silently skip scanned PDFs. Record their filenames in the evidence bundle so the lawyer sees them in the confirmation step. Defer outcome detail (what the endorsement says, what was decided) to Gmail, the matter brief (`_matter-brief.md`), or the lawyer's memory.
4. **Read `_matter-brief.md`** if present in the matter folder. The brief is a privileged current-state snapshot. Its "Open Items" and "Last Updated" sections are often decisive evidence of what was pending vs. resolved.
5. For each overdue item on this matter, build the evidence bundle:
   - **Resolved evidence**: events in Gmail / folder / brief that clearly indicate the deadline was met or the underlying task is complete. Examples:
     - Settlement conference date passed, and there's a follow-up email from opposing counsel referencing what was discussed at conference. Resolved (happened).
     - Court deadline "Serve defendants by 2026-04-10" and folder has "Affidavit of Service - 2026-04-08.pdf". Resolved.
     - Next Action "2026-03-19: 7-day cure period expires" and Gmail shows a settlement agreement signed 2026-03-18. Resolved (mooted by settlement).
     - Overdue limitation and an issued claim is in the folder (file starts with court file number or contains "Issued" / "Form 7A"). Likely resolved (claim was filed); flag for "claim filed" confirmation.
   - **Unresolved evidence**: no activity, no reference, or explicit signs the task wasn't done. Example: "Serve Form 1B on defendants by 2026-04-10" with no affidavit of service, no email about service, and no acknowledgment from opposing counsel. Unresolved.
   - **Ambiguous**: unclear. Default to treating as unresolved (safer) and flag it for the lawyer to decide.

**Scaled limitation pattern.** If the scan produced ≥5 overdue limitations AND the investigation evidence for those matters shows a court file number or issued claim in Gmail/folder (meaning the claim was clearly filed), present them together in the confirmation step as a bulk batch: "Found N limitation deadlines that appear resolved (claim filed in all cases based on court file numbers in Gmail). Review and bulk-approve?" Still gate on the lawyer confirmation. Never auto-clear a limitation. But one confirmation for N items beats N confirmations.

### Step 3: Per-Item Confirmation (one at a time)

For each overdue item, present it to the lawyer and ask for a decision.

**Use AskUserQuestion.** Group up to 4 items per tool call where possible (e.g., all 4 overdue items on the same matter), since AskUserQuestion supports 1 to 4 questions per call. If items span different matters, it's still fine to group across matters. Never exceed 4 questions per call.

Per-item question template:

```
Question text: "[File #X] [Client], [overdue_type]: '[overdue_text]' (was [overdue_date], [N] days ago). Evidence: [one-line summary of what Gmail/folder show]. Resolve?"

Header: "File #X" (or a short client tag if file # is long)

Options:
  1. "Resolve: [new Next Action or 'remove deadline']", description: "[2-3 sentence evidence summary + what will be written]"
  2. "Keep pending (unresolved)", description: "No clear evidence of completion. Will go on the red-flag list with suggested next action."
  3. "Skip this one", description: "Don't update the tracker either way. I'll revisit manually."
```

**For limitation deadlines specifically** (overdue_type = "limitation"), use a different option set:

```
Options:
  1. "Claim filed, clear limitation", description: "Will remove discovery date, statute, and deadline columns (P/Q/R). Requires evidence the claim was filed (e.g., issued claim in folder, court file number in emails)."
  2. "Close the matter", description: "The claim wasn't filed and the limitation has now expired. Closing may be appropriate if no action is intended. Will not auto-close; I'll flag it for you to run 'close matter [name]' after review."
  3. "Keep pending (unresolved)", description: "Flag as a live issue. Highest-priority red flag."
  4. "Skip this one", description: "Don't touch it. Review manually."
```

**Never** auto-clear a limitation deadline without the lawyer confirming option 1. Never auto-close a matter from this skill. If the lawyer picks option 2 on a limitation, the matter goes to the red-flag list with "run close matter [name]" as the suggested action.

### Step 4: Collect Decisions

Track each item's outcome in an in-memory structure:

```python
decisions = [
    {"file_no": "2026-012", "type": "court_deadline", "index": 0, "action": "resolve",
     "timeline_entry": "Served Form 1B on defendants", "date": "2026-04-08"},
    {"file_no": "2026-012", "type": "next_action", "action": "resolve",
     "new_next_action": "2026-05-15: Motion for summary judgment",
     "timeline_entry": "Settlement conference held; no settlement reached"},
    {"file_no": "2026-008", "type": "limitation", "action": "unresolved",
     "flag_reason": "Limitation expired 14 days ago; no claim filed. Potential malpractice exposure."},
    {"file_no": "2026-019", "type": "next_action", "action": "skip"},
    ...
]
```

### Step 5: Present Batch Summary Before Writing

Before making any write, show the lawyer exactly what's about to change:

```
Ready to batch-apply changes. Here's what I'll write:

RESOLVED (N items):
  * File #2026-012 (Patel): remove court deadline "Serve Form 1B" (2026-04-10)
    Timeline += "2026-04-08: Served Form 1B on defendants"
  * File #2026-012 (Patel): update Next Action to "2026-05-15: Motion for summary judgment"
    Timeline += "2026-03-27: Settlement conference held; no settlement reached"
  * File #2026-015 (Acme Group): remove court deadline "Amend claim" (2026-03-25)
    Timeline += "2026-03-20: Amended claim served per endorsement"
  ...

SKIPPED (X items):
  * File #2026-019 (Lee): Next Action "2026-03-15: Send draft SPA", skipped for manual review

UNRESOLVED (Y items), will go on red-flag list, no tracker writes:
  * File #2026-008 (Effa): Limitation expired 2026-04-05 (14 days ago), HIGH RISK
  * File #2026-021 (Nguyen): Next Action "2026-03-10: File defence", 40 days overdue
  ...

Apply? [Y/N]
```

**Wait for explicit "yes" / "apply" / "go" before writing.** If the lawyer wants to edit any decision, roll back to Step 3 for that item.

### Step 6: Batched Write

Once the lawyer confirms, apply the approved decisions as a **sequence of tracker_write.py calls** (see "Tracker Writes" above), in file-number order — all of a matter's items together, matching the Step 5 summary:

1. **Restore point.** Each guard call makes its own timestamped backup in `backups/`; the FIRST call's backup is the restore point for the whole sweep. Capture its path from the first call's output and cite it in the final report.
2. If the first call fails on the Excel lock file, stop and tell the lawyer to close Excel.
3. Per approved decision (the guard bumps Last Activity automatically — `timeline` sets it to max(current, event date), never a future date):

   **`action: "resolve"` on `type: "court_deadline"`:**
   - `court-deadline remove --file-no N --index I` — removes the entry and keeps the JSON structure and sort order intact.
   - `timeline --file-no N --date <date from the decision, the actual event date, not today> --text "[timeline_entry]"`.

   **`action: "resolve"` on `type: "next_action"`:**
   - `update --file-no N --set "Next Action / Deadline=[new_next_action]"`. If the user indicated there's no new next action, set the next procedural step in prose (e.g., "Awaiting client instructions re: next steps").
   - `timeline` call for the entry.

   **`action: "resolve"` on `type: "limitation"` with subtype "claim_filed":**
   - `update --file-no N --set "Discovery Date=" --set "Limitation Statute=" --set "Limitation Deadline="` (empty values clear P/Q/R).
   - `timeline --file-no N --date <issuance date from evidence> --text "Claim filed; limitation period closed."`

   **Anchored entry the lawyer resolved with a date:** `court-deadline resolve --file-no N --index I --date YYYY-MM-DD`.

   **`action: "unresolved"` or `"skip"`:** no tracker write for this item.

4. **A failed call stops the sweep at that item.** Non-zero exit = that write did not land (everything before it did). Report the stderr, what was applied, and what remains unapplied — do not continue past the failure and never retry with direct openpyxl.
5. No separate verify or validator step — each guard call saves atomically and runs validate_tracker.py itself. The guard writes values into existing cells only, so row formatting is untouched.

### Step 7: Calendar Sync

After the save, invoke `calendar-sync.reconcile(row)` for every matter that had at least one tracker write. This prunes events for removed court deadlines, updates events for changed Next Actions, and cancels the limitation event if a claim was filed. Report:

```
Calendar sync: N events removed, M updated.
```

If calendar-sync fails, log and continue. Don't roll back the tracker write.

### Step 8: Red-Flag Report

Surface unresolved items loudly. Format:

```
===============================================================
RED-FLAG LIST: Items still pending after triage
===============================================================

LIMITATION EXPIRED (HIGHEST PRIORITY):
  * File #2026-008: Effa, M.
    Matter: Constructive dismissal claim
    Limitation expired: 2026-04-05 (14 days ago)
    Last activity: 2026-03-01 (49 days ago)
    Statute: general_statute
    > Suggested action: URGENT. Confirm whether claim was filed. If not, assess malpractice exposure and notify insurer if applicable. Do NOT file out of time without a discoverability analysis under your jurisdiction's limitation statute.

COURT DEADLINES PAST DATE:
  * File #2026-019: Lee, N.
    Deadline: "Serve Form 1B on defendants", was 2026-04-10 (9 days ago)
    Last activity: 2026-03-15 (35 days ago)
    > Suggested action: Confirm service status. If not served, serve immediately and disclose late service to the court. If served, run "update matter Lee" to log the service date.

NEXT ACTION PAST DATE:
  * File #2026-021: Nguyen, T.
    Next Action: "2026-03-10: File defence"
    Overdue by: 40 days
    Last activity: 2026-02-20 (58 days ago)
    > Suggested action: Confirm whether defence was filed. If not, check for default proceedings. If filed, update the tracker.

STALE BUT NOT OVERDUE:
  (none. This skill only reports overdue. Run 'show my open files' with a staleness filter to see non-overdue stale matters.)

===============================================================
```

**Recurrence escalation.** Persist sweep results to `backups/overdue-triage-history.json` beside the tracker (same folder as the batch backup): for each unresolved item, append `file_no`, item text, and sweep date. On each sweep, cross-check the new unresolved list against this history. Items flagged unresolved in 2+ prior sweeps get an escalation line in the red-flag report instead of a plain repeat: "flagged N times since [date]: pattern suggests a decision blocker. Recommend call client / close matter / fix the tracker entry."

**Suggested action writing rules:**
- Be specific. "Follow up" is not a suggestion. Name who to contact, what to check, and what the next procedural step is.
- For limitation expiries, always flag malpractice exposure analysis. Never minimize.
- For service deadlines, always mention any relief-from-consequences option your rules of procedure provide if the deadline was missed.
- For court deadlines tied to endorsements, suggest checking the order text to see if the court attached consequences (e.g., "failure to serve results in dismissal").
- Match the lawyer's preference: no em dashes, no sugar-coating.

## Behaviour Rules

1. **The first backup of the sweep is the batch restore point.** Each guard call backs up on its own; cite the first call's backup path in the final report. A failed call stops the sweep at that item — report what was applied and what remains.
2. **Never auto-clear limitation deadlines.** Always require the lawyer's explicit "Claim filed" confirmation.
3. **Never auto-close matters.** If a matter looks closeable, add it to the red-flag list with "run close matter [name]" as the suggested action.
4. **Always show the batch summary before writing.** No silent writes.
5. **Confirm one item at a time, up to 4 per AskUserQuestion call.** Group items on the same matter where possible to reduce round-trips, but per-item granularity is the user's explicit requirement.
6. **If Gmail is unavailable, proceed with folder-only investigation and warn the lawyer up front.** Evidence will be weaker; expect more "unresolved" outcomes.
7. **If the matter folder (column T) is blank**, skip the folder scan for that matter and rely on Gmail alone. Don't try to fuzzy-resolve the folder inside this skill. That belongs to matter-tracker.
8. **Ambiguous evidence defaults to "unresolved".** When in doubt, flag rather than auto-resolve. This is a cleanup skill, not an inference skill.
9. **Timeline entries carry the actual event date.** Append via `timeline` with the event date, not today. If column J already holds an entry with the same date and substance, skip it (don't duplicate).
10. **Report the full numbers at the end.** "Scanned N matters, N1 overdue items, N2 resolved, N3 flagged, N4 skipped."
11. **Stop and ask if scan detects zero overdue items.** Say "Tracker is clean, no overdue items. Nothing to do." Do not proceed to investigation.
12. **Court deadlines index matters.** `court-deadline remove` takes the 0-based index of the array as currently stored. When removing multiple entries on the same matter, remove the highest index first — and since the guard re-sorts the array on every mutation, re-read column S between calls if there is any doubt about the surviving indices.
13. **Group same-date events across columns I and S.** A hearing listed in both columns (same date, overlapping description) is one event. Confirm once, write to both columns.
14. **Investigate in batches of 8 matters.** Emit a progress line between batches. Persist investigated evidence to `/tmp/overdue-triage-session.json` so an interrupted run can resume without re-pulling Gmail.
15. **Scanned PDFs count as evidence.** Filename + modification date on a court endorsement or affidavit-of-service PDF is sufficient to establish occurrence even when text extraction fails. Record the filename in the evidence bundle.
16. **Data-quality warnings go at the top of the output**, before the red-flag list. Non-date Last Activity values, invalid Court Deadlines JSON, and orphaned limitation statute/deadline pairs all belong here. This is a tracker-hygiene skill by side effect, so surface the problems rather than working around them silently.

## Suggested-Action Library

A concise reference for writing the red-flag list. Use these as templates. Customize to the specific facts.

| Scenario | Template suggested action |
|----------|--------------------------|
| Limitation expired, no claim filed | "URGENT: Limitation expired [N] days ago. Confirm non-filing. If confirmed, conduct a discoverability analysis under your jurisdiction's limitation statute for any late-filing argument. Notify insurer if exposure exists." |
| Court deadline missed (service) | "Confirm service status. If not served, serve immediately. Assess whether relief from the missed deadline is available under your rules of procedure. Notify opposing counsel of late service and confirm no prejudice." |
| Court deadline missed (filing) | "Confirm filing status. If not filed, file now with an explanation letter to the court. Check for any default proceedings initiated by opposing counsel." |
| Court deadline missed (endorsement compliance) | "Check the endorsement text for automatic consequences. If the endorsement attached consequences (e.g., dismissal, striking of claim), assess rule 37.14 motion to set aside. Notify client immediately." |
| Next Action: settlement conference passed | "Log conference outcome (result, positions taken, next steps). If ordered to a further step, add to court deadlines. If settled, proceed to close." |
| Next Action: cure period expired | "Confirm whether breach was cured. If cured, note and proceed. If not, file for judgment per the agreement. Client decision needed." |
| Next Action: client instructions overdue | "Follow up with client directly (phone, not email). If no response after [7 days], send formal written follow-up. If still no response, consider terminating the retainer per the engagement letter." |
| Next Action: waiting on opposing counsel | "Follow up with opposing counsel by letter (not email). If no response in [7 days], proceed with the next procedural step unilaterally." |
| Generic missed deadline | "Confirm current status via client/file review. Update Next Action to reflect actual state. If the deadline created a cascade (e.g., defence due triggers disclosure), recalculate downstream dates." |

## Output Checklist (at end of run)

Every overdue-triage run ends with a final message that includes:

1. Scan summary (N matters scanned, K overdue items found).
2. Batch write result (N1 resolved; first backup path = restore point for the whole batch; if a call failed, what was applied and what remains).
3. Calendar sync diff (N events removed, M updated).
4. Red-flag list in full (every unresolved item with suggested action).
5. Any errors or warnings (Gmail unavailable, folder not found, etc.).
6. One-line closer: "Overdue triage complete. Next recommended run: in ~3-4 weeks."
