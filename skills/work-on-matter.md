---
name: work-on-matter
description: "Use this skill whenever the user wants to resume work on an existing matter or client file. Trigger on phrases like 'let's work on matter [name]', 'let's work on [name]', 'pull up [name]', 'open the [name] file', 'where are we with [name]', 'bring yourself up to speed on [name]', or any request to pick up, continue, or revisit a client matter. Also trigger on 'what do we have on [name]' or 'refresh yourself on [name]'. Do NOT trigger on 'new matter', 'update matter', or 'close matter' — those belong to the matter-tracker skill. This skill loads context and does work on a matter, with lightweight tracker updates inline as work gets done."
---

# Work on Matter — Context Loader

## Purpose

This skill loads context for an existing matter so you can pick up where you left off in a new chat session. It reads from the matter tracker spreadsheet and a per-matter brief file (`_matter-brief.md`) stored in the client's matter folder -- which may be the top-level client folder or a matter-specific subfolder within it. The goal is to orient yourself quickly without re-reviewing underlying documents. As you do substantive work, this skill also does lightweight inline updates to the tracker spreadsheet (Last Activity, Timeline, Next Action) so the tracker stays current without requiring a separate "update matter" step.

## When This Runs

The user says something like "let's work on matter Smith" or "pull up the Chen file." They want you oriented and ready to answer questions or do work on that matter.

## Conventions

- **"the lawyer"** refers to the user.
- **"Client"** refers to the person/entity who retained the lawyer on the matter.

## Dependencies

- **Matter tracker spreadsheet**: The tracker (`matter-tracker.xlsx`) lives in the Open Files directory alongside the matter folders. To find it: check the current working directory first, then the parent directory, then one level up. If not found after three checks, ask the user for the path. Do not glob recursively from the home directory.
- **Filesystem access**: The Open Files directory contains both the tracker and all matter folders as sibling subdirectories. The user may set the working directory to the Open Files parent, or to a specific matter folder.

## Workflow

### Step 1 — Find the Matter in the Tracker

1. Extract the client/matter name from the user's message.
2. Load the matter tracker spreadsheet. Check CWD, then CWD's parent, then one level up. If not found after three checks, ask the user for the path. Search the "Open Matters" sheet for a matching row (case-insensitive partial match on Client Name, Matter Description, or Opposing Party). Also check "Closed Matters" if no match found on Open.
3. If multiple matches, ask the user to clarify.
4. If no match found on either sheet, tell the user there's no existing file for that name. Ask whether they want to open a new matter (which will trigger the matter-tracker skill's "new matter" workflow) or if they may have the name wrong.

### Step 2 — Resolve the Matter Folder (and Subfolder)

Client folders often contain subfolders for separate matters (e.g. "Real Estate Purchase/", "Small Claims - Damage Deposit/", "Incorporation/"). This step resolves the correct folder -- which might be the client folder itself or a matter-specific subfolder within it -- so the brief is read from and saved to the right place.

1. From the matching row, get the Matter Folder name (column T).
2. **If column T has a value**: The matter folder is a sibling directory of the tracker file. List the subdirectories in the same directory as `matter-tracker.xlsx` and find the one matching column T. This is the **client folder**: `<open-files-dir>/<matter-folder>/`.
3. **If column T is blank**: List all sibling directories of the tracker file and do a case-insensitive fuzzy match against the client name -- try last name, first name, full name, entity name, and permutations. If a match is found, use it (and note to the user that the tracker's Matter Folder column is blank and should be updated). If no match, proceed without the brief.
4. **Check for matter-specific subfolders.** List the immediate contents of the client folder. If there are subdirectories:
   - Check whether any subfolder name matches the matter description (column C) or opposing party (column H) using case-insensitive keyword matching. Subfolders are often named after the deal type ("Incorporation/", "Lease/"), the dispute ("Smith v. Jones/"), or a short description ("Small Claims - Damage Deposit/").
   - If a matching subfolder is found, that subfolder is the **resolved matter folder**. Look for `_matter-brief.md` there.
   - If no subfolder matches but `_matter-brief.md` exists at the client folder's top level, use the client folder as the resolved matter folder.
   - If the client folder has subfolders, none match, and no brief exists at the top level either, ask the user which subfolder this matter lives in.
   - If the client folder has no subdirectories, the client folder itself is the resolved matter folder.
5. Look for `_matter-brief.md` in the resolved matter folder.
6. If the brief exists, read it.
7. If the brief doesn't exist, note this -- it's fine, this might be the first time using this workflow for this matter.
8. If you can't find the client folder at all, proceed without it -- you can still orient from the tracker data alone. Flag that the brief can't be read or written until the folder is identified.

**Remember the resolved matter folder path** -- you'll use it in Step 4 when saving the brief.

### Step 3 — Orient and Summarize

Present a concise summary to the user:

```
Matter: 2026-XXX | [Client Name]
Description: [Matter Description]
Status: [Status]
Last Activity: [Date]
Next Action: [Next Action / Deadline]

[If brief exists and is current (Last Updated >= Last Activity):]
From the matter brief:
- [Key points from the brief — parties, deal summary, flagged risks, open items]

[If brief exists but is stale (Last Updated < Last Activity from tracker):]
From the matter brief (last updated [brief date] — tracker shows activity on [tracker date], brief may be outdated):
- [Key points from the brief]

[If brief exists but Last Updated date can't be parsed (e.g., hand-edited, non-standard format):]
From the matter brief (last updated date unclear — treating as potentially stale):
- [Key points from the brief]

[If no brief exists but tracker has a Timeline (column J):]
No matter brief on file yet. Here's the timeline from the tracker:
[Timeline entries from column J]
I'll start a brief once we do substantive work.

[If no brief exists and no timeline:]
No matter brief on file yet and no timeline in the tracker. I'll start one once we do substantive work.

[If limitation deadline exists and is within 6 months:]
Limitation deadline: [date] — [X days remaining]

[If court deadlines exist:]
Upcoming court deadlines: [list any within 60 days]
```

Then: "Ready to go. What are we working on?"

### Step 4 — Do the Work (and Save As You Go)

Proceed with whatever the user needs — review documents, draft things, answer questions, etc.

**CRITICAL: Every substantive task has three parts — (1) do the work, (2) save the brief, (3) update the tracker. A task is not complete until both `_matter-brief.md` and the tracker spreadsheet are updated. Do all three in the same response. Never plan to "save later" or "save at the end." There is no end — sessions crash, compact, or just stop.**

#### Citation Discipline

Legal work lives and dies on specific citations. A section number you recall from a similar lease you never actually opened, or a date you inferred from context, is the kind of error that destroys client trust and creates real liability. Treat your memory as a prompt, not a source.

Before any of the following appear in client-facing output (emails, letters, opinions, redlines, advice memos, tracker Timeline entries):

- Section numbers and clause references (e.g. s.10.1, Article 11, Schedule B)
- Dollar figures and dates
- Party names, entity numbers, property addresses
- Quoted or paraphrased clause text

...open the source document in the matter folder and confirm the citation matches what's actually there. Do this even when you're confident. Confidence is not the signal; it's often what produces the error. A quick Read on the relevant page of the lease or agreement takes seconds; a wrong citation in a client email costs much more.

If the source isn't available in the folder (missing page, document not provided, redacted version, etc.), do not cite from memory. Flag the gap to the user and either request the document or frame the advice without the citation. "Landlord's consent is required under the assignment provision" is always better than "landlord's consent is required under s.10.1" when you haven't actually read s.10.1.

This rule is narrower and more enforceable than "always double-check." The point is not to verify things in general, which produces theatre. It's to catch the specific failure mode where a citation sounds right but isn't, because the model is reaching for something plausible instead of looking at the page.

After completing any of these, immediately update both the brief and the tracker in the same response:

- You reviewed a document and formed conclusions
- You gave material advice or flagged a risk
- You drafted something (letter, clause, memo, etc.)
- A strategic decision was made
- The user explicitly says "save that," "update the tracker," or similar

Do NOT save after quick factual lookups (e.g. "what's the limitation deadline?").

**How to save the brief:**

1. Read the existing `_matter-brief.md` (if it exists) from the **resolved matter folder** determined in Step 2. This might be a matter-specific subfolder (e.g. `<client-folder>/Incorporation/`) or the client folder itself -- use whichever path Step 2 resolved.
2. Merge in what just happened -- update relevant sections, replace stale info, add new items. Keep it a current-state snapshot, not a running log.
3. Write the updated brief back to the same resolved matter folder.

**If no brief exists**, create one from scratch based on what you know from the tracker, documents reviewed, and work done. Save it to the resolved matter folder.

**If you couldn't resolve the matter folder path**, save the brief to the same directory as the tracker file with the filename `_matter-brief-[client-name].md` and tell the user to move it manually.

**How to update the tracker (lightweight inline write):**

This is NOT a full tracker refresh -- no Gmail scan, no folder audit. Just three targeted cell updates on the matter's row in the tracker spreadsheet:

1. **Last Activity (column F):** Set to today's date.
2. **Timeline (column J):** Append a one-line entry for what was done. Format: `YYYY-MM-DD -- [brief description]`. Append with a newline after existing content -- never overwrite prior timeline entries.
3. **Next Action (column G):** Update only if the work changes what's next. If the existing Next Action is still correct, leave it alone.

**Before writing: back up the tracker.** Copy `matter-tracker.xlsx` to `backups/matter-tracker-backup-YYYY-MM-DD.xlsx` in a `backups/` subfolder alongside the tracker. Create the `backups/` folder if it doesn't exist. After the write, re-open the tracker with openpyxl to confirm it loads cleanly. Do NOT delete older backups -- let the folder accumulate history. If verification fails, tell the user the write may have corrupted the file and point them to the most recent backup in `backups/`. This matters because sessions can crash mid-write and Excel files corrupt silently -- backups are the only safety net, and silent corruption can go unnoticed for days, so keeping history matters.

Use the xlsx skill's openpyxl approach to read, modify, and save the tracker. Keep the row reference from Step 1 so you don't need to re-search.

**If the tracker can't be written** (permissions, file locked, etc.), don't let it block the user's work. Flag it once ("Couldn't update the tracker -- file may be open elsewhere") and continue. The brief still captures the session context.

#### Brief Format

The brief should stay **short — one page max**. It's a current-state snapshot, not a diary. Think of it like a reporting letter to a client: if a new associate picked this up tomorrow, what do they need to know?

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

Omit any section that doesn't apply (e.g., skip "Key Terms" for a litigation matter). The point is brevity — if the brief is getting long, you're putting in too much detail.

**Privilege warning**: Always include this header at the top of every brief:

```
> PRIVILEGED & CONFIDENTIAL — Solicitor-Client Privilege / Work Product
```

After the **first** save in a session, let the user know: "Matter brief and tracker updated." Subsequent saves should be silent -- don't announce every update. If the user asks, confirm both the brief and tracker are being kept current.

## Important Rules

1. **Save inline, not later.** After every substantive task (document review, advice, drafting, decision), update both `_matter-brief.md` and the tracker in the same response as the work. Never defer to a "wrap-up" step -- sessions end without warning. This is the single most important rule in this skill.
2. **Tracker writes are lightweight only.** This skill updates Last Activity, Timeline (append), and Next Action. It does NOT do Gmail pulls, folder scans, or full tracker refreshes -- that's the matter-tracker skill's "update matter" workflow. If the user needs a comprehensive refresh, they should run "update matter [name]."
3. **Don't run a Gmail pull or full folder scan.** This skill is for fast context loading and inline work, not research.
4. **Keep the brief lean.** If you find yourself writing more than ~40 lines, you're including too much. Summarize; don't transcribe.
5. **Don't save after quick lookups.** "What's the limitation deadline?" doesn't warrant a brief or tracker update.
6. **Find the tracker automatically.** Check CWD, then CWD's parent, then one level up. If not found after three checks, ask the user. Do not glob recursively.
7. **Don't double-write the brief.** If the matter-tracker skill already wrote/refreshed `_matter-brief.md` during this session (e.g., the user ran "update matter [name]"), skip the brief save in this skill to avoid overwriting the tracker skill's more comprehensive output. The tracker timeline append is still fine -- duplicating a timeline entry is harmless compared to losing one.
8. **Back up the tracker before every write, into `backups/`.** Never write to the tracker without first copying it to `backups/matter-tracker-backup-YYYY-MM-DD.xlsx`. Backups live in the `backups/` subfolder alongside the tracker, never in the Open Files root. After the write, verify the tracker still opens with openpyxl; if it doesn't, point the user to the most recent backup in `backups/`. Never auto-delete older backups -- silent corruption can go undetected for days, so keep the full history.
9. **Verify citations from source.** Any section number, dollar figure, date, party name, or quoted clause text that appears in client-facing output (emails, letters, redlines, advice, tracker entries) gets confirmed against the source document in the matter folder before it's written. If the source isn't available, flag the gap and don't cite. This is the most common failure mode for legal work: a cite that sounds right but isn't, because it was reached for from memory instead of pulled from the page.
