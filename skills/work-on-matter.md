---
name: work-on-matter
description: "Use this skill whenever the user wants to resume work on an existing matter or client file. Trigger on phrases like 'let's work on matter [name]', 'let's work on [name]', 'pull up [name]', 'open the [name] file', 'where are we with [name]', 'bring yourself up to speed on [name]', or any request to pick up, continue, or revisit a client matter. Also trigger on 'what do we have on [name]' or 'refresh yourself on [name]'. Do NOT trigger on 'new matter', 'update matter', or 'close matter' — those belong to the matter-tracker skill. This skill is for loading context at the start of a work session, not for modifying the tracker spreadsheet."
---

# Work on Matter — Context Loader

## Purpose

This skill loads context for an existing matter so you can pick up where you left off in a new chat session. It reads from the matter tracker spreadsheet and a per-matter brief file (`_matter-brief.md`) stored in the client's folder. The goal is to orient yourself quickly without re-reviewing underlying documents.

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

### Step 2 — Read the Matter Brief

1. From the matching row, get the Matter Folder name (column T).
2. **If column T has a value**: The matter folder is a sibling directory of the tracker file. List the subdirectories in the same directory as `matter-tracker.xlsx` and find the one matching column T. The brief lives at `<open-files-dir>/<matter-folder>/_matter-brief.md`.
3. **If column T is blank**: List all sibling directories of the tracker file and do a case-insensitive fuzzy match against the client name — try last name, first name, full name, entity name, and permutations. If a match is found, use it (and note to the user that the tracker's Matter Folder column is blank and should be updated). If no match, proceed without the brief.
4. Look for `_matter-brief.md` in the resolved folder.
5. If the brief exists, read it.
6. If the brief doesn't exist, note this — it's fine, this might be the first time using this workflow for this matter.
7. If you can't find the matter folder at all, proceed without it — you can still orient from the tracker data alone. Flag that the brief can't be read or written until the folder is identified.

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

**CRITICAL: Every substantive task has two parts — (1) do the work, (2) save the brief. A task is not complete until `_matter-brief.md` is written/updated. Do both in the same response. Never plan to "save later" or "save at the end." There is no end — sessions crash, compact, or just stop.**

After completing any of these, immediately update `_matter-brief.md` in the same response:

- You reviewed a document and formed conclusions
- You gave material advice or flagged a risk
- You drafted something (letter, clause, memo, etc.)
- A strategic decision was made
- The user explicitly says "save that," "update the tracker," or similar

Do NOT save the brief after quick factual lookups (e.g. "what's the limitation deadline?").

**How to save:**

1. Read the existing `_matter-brief.md` (if it exists).
2. Merge in what just happened — update relevant sections, replace stale info, add new items. Keep it a current-state snapshot, not a running log.
3. Write the updated brief back as a coherent document.

**If no brief exists**, create one from scratch based on what you know from the tracker, documents reviewed, and work done.

**If you couldn't resolve the matter folder path**, save the brief to the same directory as the tracker file with the filename `_matter-brief-[client-name].md` and tell the user to move it manually.

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

After the **first** save in a session, let the user know: "Matter brief saved to [folder]/_matter-brief.md." Subsequent incremental saves should be silent — don't announce every update. If the user asks, confirm the brief is being kept current.

If the work session involved events that belong on the timeline (e.g., you drafted a document, reviewed a contract, gave substantive advice, or a key decision was made), proactively suggest: "This session had substantive updates. Want me to run 'update matter [name]' to refresh the tracker timeline?"

If the user declines or it's not relevant, that's fine — the brief alone captures the session context.

## Important Rules

1. **Save the brief inline, not later.** After every substantive task (document review, advice, drafting, decision), update `_matter-brief.md` in the same response as the work. Never defer it to a "wrap-up" step — sessions end without warning. This is the single most important rule in this skill.
2. **This skill is read-heavy, write-light.** The only file it creates/modifies is `_matter-brief.md`. It does not touch the tracker spreadsheet — that's the matter-tracker skill's job.
3. **Don't run a Gmail pull or full folder scan.** This skill is for fast context loading, not research. If the user needs a full timeline refresh with Gmail, they should run "update matter [name]" via the tracker skill.
4. **Keep the brief lean.** If you find yourself writing more than ~40 lines, you're including too much. Summarize; don't transcribe.
5. **Don't save after quick lookups.** "What's the limitation deadline?" doesn't warrant a brief update.
6. **Find the tracker automatically.** Check CWD, then CWD's parent, then one level up. If not found after three checks, ask the user. Do not glob recursively.
7. **Don't double-write the brief.** If the matter-tracker skill already wrote/refreshed `_matter-brief.md` during this session (e.g., the user ran "update matter [name]"), skip the brief save in this skill to avoid overwriting the tracker skill's more comprehensive output.
