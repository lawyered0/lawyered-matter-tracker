---
name: work-on-matter
description: "Use this skill whenever the user wants to resume work on an existing matter or client file. Trigger on phrases like 'let's work on matter [name]', 'let's work on [name]', 'pull up [name]', 'open the [name] file', 'where are we with [name]', 'bring yourself up to speed on [name]', or any request to pick up, continue, or revisit a client matter. Also trigger on 'what do we have on [name]' or 'refresh yourself on [name]'. Do NOT trigger on 'new matter', 'update matter', or 'close matter' — those belong to the matter-tracker skill. This skill loads context from the tracker plus three per-matter files (`_matter-brief.md`, `_matter-decisions.md`, `_matter-comms.md`), ALWAYS runs a bounded Gmail pull on the past 7 days at the start of every session and refreshes the brief with any material findings before orienting, and does inline saves as work progresses."
---

# Work on Matter — Context Loader

## Purpose

Load context for an existing matter so you can pick up mid-stream in a new session. Reads the matter tracker plus up to three per-matter files:

- `_matter-brief.md` — current-state snapshot
- `_matter-decisions.md` — append-only strategic decisions log
- `_matter-comms.md` — append-only file-specific communications / client preferences

Every load runs a bounded Gmail pull and merges material findings into the brief BEFORE orientation. Saves inline as work happens.

## Conventions

- **the lawyer** = the user.
- **Client** = the person/entity who retained the lawyer.

## Dependencies

- **matter-tracker.xlsx**: lives in the Open Files directory alongside matter folders. Check CWD, then parent, then one level up. If not found after three checks, ask the user. Don't glob recursively from home.
- **Filesystem access** to the Open Files directory (tracker + sibling matter folders).
- **calendar-sync skill**: invoked after the tracker write when Next Action changes to a new dated value or work surfaces a dated third-party follow-up. If unavailable, skip and note it once — same rule as the Calendar Sync Hook section; never fail silently, and never block the work.
- **Gmail MCP tools**: `search_threads` and `get_thread`. Required for Step 2.5. If unavailable, surface the gap in Step 3 — never proceed silently as if the brief is fresh.

## Workflow

### Step 1 — Find the Matter in the Tracker

1. Extract the client/matter name from the user's message.
2. Load the tracker. Search "Open Matters" sheet for a matching row (case-insensitive partial match on Client Name, Matter Description, or Opposing Party). Also check "Closed Matters" if no match.
3. **If multiple matches, present a numbered chooser before doing anything else.** Use AskUserQuestion if available.
   ```
   Multiple matches for "[name]". Which one?
   1. 2026-XXX | [Client Name] — [Matter Description] (Last Activity [date])
   2. 2026-YYY | [Client Name] — [Matter Description] (Last Activity [date]) [CLOSED]
   ```
   Sort open first, then closed. Mark closed with [CLOSED]. Wait for the answer.
4. If no match anywhere, tell the user. Ask whether they want to open a new matter or if they have the name wrong. Suggest close matches (e.g., "Persuad" → "Persaud").
5. **If the selected matter is CLOSED, load it read-only.** Run Steps 2 and 3 to orient, but skip Step 2.5's brief/tracker writes and make NO writes to matter files, the tracker, or the calendar. Before any substantive work, ask the user to reopen first ("reopen matter [name]", handled by matter-tracker). The reason: inline saves would put fresh activity on the Closed Matters sheet and the calendar hook could push events for a file that is supposed to have zero — calendar-sync's rule is "closed matters have zero events." Answering questions from the file is fine; changing the file is not.

### Step 2 — Resolve the Matter Folder (and Subfolder)

Client folders often contain subfolders for separate matters. Resolve the correct folder so files are read from and saved to the right place.

1. Get Matter Folder name from column T.
2. **If column T has a value**: find the sibling directory of the tracker matching column T. That's the **client folder**.
3. **If column T is blank**: fuzzy-match sibling directories against the client name (last, first, full, entity, permutations). Note that the tracker should be updated. If no match, proceed without the folder.
4. **Check for matter-specific subfolders.** List the client folder's immediate contents.
   - Match subfolder name against column C (description) or column H (opposing party) by keyword.
   - If a subfolder matches → that's the **resolved matter folder**.
   - If none match but `_matter-brief.md` exists at the client folder's top level → use the client folder.
   - If subfolders exist, none match, and no top-level brief → ask the user which subfolder.
   - If no subdirectories → the client folder is the resolved matter folder.
5. Look for the **three matter files** in the resolved folder. Read each that exists. **Record each file's mtime when first read** — needed in Step 4 for the concurrent-session check.
6. If a file doesn't exist, fine — files are created on first need.
7. If the client folder can't be found, proceed from tracker data alone. Flag that matter files can't be read or written until the folder is identified.

Remember the resolved folder path and the mtimes — both are needed in Step 4.

### Step 2.5 — Email Refresh (ALWAYS run; updates the brief before orientation)

**This step always runs on every load. No conditionals.** A brief that's even a few days old will silently miss anything that came in since. The cost is one Gmail search; the cost of skipping is a confidently wrong orientation.

Not a full update — no folder scan, no full timeline rebuild, no per-email timeline entries. A targeted pull on a fixed window, brief refresh if anything material, then orient from the refreshed brief. If material findings emerge, the refresh ends with ONE combined tracker entry (see the brief-refresh steps below) — that single write is expected and is not a violation of this rule.

**Lookback window:**

- Default: **past 7 days**, always, regardless of recency.
- If Last Activity (column G) or the brief's `## Last Updated` is older than 7 days, extend to that date minus 1 day.
- Cap at **30 days**. If the brief is months stale, tell the user "Brief is very stale — recommend running 'update matter [name]' for a full refresh" and pull 30 days.

**Three passes, ordered C → A → B** (cheapest deterministic check first). Each pass catches what the others miss. Run every pass whose inputs exist.

**Pass C — Known-thread refresh.** Catches every new message on a thread already on file, regardless of keywords or sender. The only pass that reliably catches short replies from senders not yet on file.

1. Read the brief's `## Tracked Threads` block. Each line stores a Gmail thread ID, a short subject label, and the date of the most recent processed message.
2. For each tracked thread ID, call `get_thread` with `messageFormat=FULL_CONTENT` — this is what makes escalation a no-op for Pass C results. Identify messages dated after the thread's "last seen" date (or after `## Last Updated` if no per-thread date).
3. Feed new messages into the triage logic below. Full bodies are already returned, so escalation is a no-op for Pass C results.
4. After triage, update the "last seen" date for each thread that produced new messages (done as part of the brief refresh, not inline mid-pull).
5. If the brief has no `## Tracked Threads` block yet, skip Pass C and note in Step 3 that thread tracking is bootstrapping. Pass A/B will seed the block on this load — and because the dedupe step (below) now calls `get_thread` on every Pass A/B thread, bootstrap loads see every message on the matter's main threads, not just the early ones the search returned.

**Pass A — Keyword pass.** Catches new threads and any message where matter-specific keywords appear in body, subject, or headers. Seeds Pass C's tracked-thread list.

1. Build the query, joined with OR:
   - **Client name** (entity AND principal, as separate OR'd terms).
   - **Opposing party** from column H if populated.
   - **Named role-holders from the brief's `## Roles` section** — opposing principals, opposing counsel, witnesses, experts, agents, paralegals, building managers. Parse names out of the role lines. Skip the client (covered) and skip generic role labels.
   - **Unusual matter-specific keywords from column C** (property address, court file number). Skip generic terms like "lease" or "claim" — they over-match.
2. Run `search_threads` with that query plus `newer_than:` set to the lookback window. **Use only the returned thread IDs — see the truncation warning below.**

**Pass B — From-address pass.** Catches short replies on existing threads where the body has no matter keywords AND the thread isn't yet in `## Tracked Threads`.

1. Collect known addresses tied to this matter:
   - Client Email (tracker column M).
   - Addresses in the brief's `## Roles` section.
   - Opposing counsel address if recorded.
   - Court / tribunal / third-party addresses that have appeared before.
2. Build: `(from:addr1 OR from:addr2 OR …) newer_than:Xd`. Same window as Pass A.
3. Run `search_threads`. **Use only the returned thread IDs — see the truncation warning below.**
4. If no addresses are known, skip Pass B and note it in Step 3.

**Combine, dedupe, and triage. Default to snippets; escalate to full read only when warranted.**

Gmail's search response gives sender, date, subject, and a ~150-char snippet. Read the minimum sufficient unit — reading every thread in full produces the failure mode this skill exists to prevent.

⚠️ **Critical: `search_threads` truncates the per-thread message list.** The response for each thread contains only a handful of matching messages, often just the earliest few. Recent messages on long-running threads are silently absent from the response even when they would match the query. This means Pass A and Pass B will systematically miss new activity on any thread that has had more than a few prior messages — exactly the threads where new activity matters most. The fix: treat `search_threads` as a **thread-ID discovery tool only**, then re-fetch each thread via `get_thread` to see the actual full message list.

1. **Collect candidate thread IDs.** Union the unique thread IDs from Pass A, Pass B, and Pass C. Pass C threads already arrived via `get_thread` with the full message list in hand; Pass A and Pass B contributed thread IDs only. Dedupe across passes — a single thread can surface from multiple passes.
2. **For every Pass A/B thread, call `get_thread`.** Use `messageFormat=MINIMAL` (snippets + headers, no bodies — same level of detail you'd get from search_threads if it weren't truncating). Skip threads already in hand from Pass C. This step is non-optional: skipping it reintroduces the truncation bug. The cost is one tool call per candidate thread, typically a handful per load.
3. **Filter to messages within the lookback window.** For Pass C threads, keep messages dated after that thread's "last seen". For Pass A/B threads (first contact with the thread, or thread not in `## Tracked Threads`), keep messages dated after `lookback_start`. Discard everything else — quoted history, old replies, and pre-window activity are noise.
4. **Decide read mode for the pull as a whole**, based on brief freshness at session start:
   - **<48 hours**: snippets-only by default. Full reads only on trigger.
   - **2–7 days**: snippets-first; escalate generously when in doubt.
   - **>7 days, OR Gmail was unavailable last load, OR active litigation/transactional crunch (court date or closing within 14 days)**: default to full reads.
5. **Per-message escalation triggers** — re-call `get_thread` with `messageFormat=FULL_CONTENT` (or, for Pass C, the bodies are already in hand) if the snippet or sender domain implies:
   - Court / tribunal / regulator domain (`@ontario.ca`, LTB, HRTO, College, etc.). Always full-read; always surface even if administrative.
   - Opposing counsel substantive content (not a one-line ack).
   - Dollar figure, date, section/clause reference, or deadline word ("by", "no later than", "within", "before").
   - New role, name not in the brief, or unknown domain.
   - Client describing instructions, decisions, settlement positions, or material facts.
   - Snippet ends mid-sentence and surrounding context is non-trivial.
   When in doubt, one full read. The cost is one tool call.
6. **Skip the full read** for scheduling acks, one-line confirms, automated receipts, forwarded marketing.
7. **For each message processed**, extract: sender, date/time, what changed. Court / tribunal emails are always reported even if administrative.

**Brief refresh.** Classify each finding:

- **Material** — affects current state: new/changed role, new risk, advice given/received, position taken, deadline set or moved, document exchanged, stage change.
- **Informational** — scheduling chitchat, "got it thanks", calendar invites.

If ANY material, refresh the brief BEFORE Step 3:

1. Merge findings into the appropriate section: Roles, Risks & Issues, Positions Taken, Open Items, Key Terms (with source citation).
2. **Update `## Tracked Threads`.** Every thread that produced findings on this load (material OR informational) gets a line with the latest message date as "last seen". New threads appended; existing entries advanced. This is what makes Pass C effective on the next load — skipping it reopens the gap.
3. Update `## Last Updated` to today.
4. Save via the **Universal Save Procedure** in Step 4. If no brief existed, create it.
5. Update Last Activity (column G) and append a single combined Timeline entry (column J) via the lightweight tracker write. Example: `2026-04-28 -- Brief refreshed from email: opposing counsel sent updated lease; client confirmed bank trail.`
6. Continue to Step 3.

If all findings are informational, still update `## Tracked Threads` for thread movement (advances "last seen" so Pass C doesn't re-process). Brief save, no tracker write. Step 3 surfaces the items in "What's new".

**If Gmail tools are unavailable:** Skip the pull. Note in Step 3 orientation: "Couldn't check Gmail — orienting from brief alone, which may be stale. Recommend running 'update matter [name]' if anything important might have come in." Never proceed silently.

### Step 3 — Orient and Summarize

A runway back into the work, not a closing memo. Order: rules of engagement first (so they bind before drafting), live story middle, deadlines and what's new last (freshest when work begins).

**Length discipline.** Short. The example below is the target length, not the floor. The brief is the source of truth; the orientation is a pointer back into it.

Skeleton (omit blocks that don't apply):

1. **Header** — one line: file number, client, description, status, last activity, next action.
2. **Comms / Preferences block** (if `_matter-comms.md` has any entries) — print verbatim, treat as binding for this session.
3. **Brief story** — three to six bullets from the live sections of `_matter-brief.md`. Pick what's actionable today, not what paints the most complete picture. If no brief, quote the tracker timeline.
4. **Recent decisions** (if `_matter-decisions.md` exists) — last three to five entries, one line each (date + headline only); full log in the file.
5. **Deadline alerts** — limitation within 6 months, court deadlines within 60 days, any explicit deadline in the next 14 days. One line each.
6. **What's new from the email pull** — one line per email/thread, prioritized by urgency, [URGENT] tag on court emails or sub-7-day deadlines. Material → "Brief refreshed from email pull. Material updates merged in:". Informational only → "Informational only (brief unchanged):". Nothing → "No new email activity in the past [N] days." Gmail unavailable → use the fallback line above.

**If the orientation exceeds ~25 lines including blanks, you're summarizing, not orienting. Cut.**

**Example orientation:**

```
Matter: 2026-148 | Marcus Feld
Description: Toronto Small Claims defence (CAM/utility dispute)
Status: Open
Last Activity: 2026-04-25
Next Action: 2026-05-03: Defence due

Comms / Preferences on file (treat as binding):
- 2026-04-23 — Written only on this file; no calls. Per the lawyer.

From the matter brief:
- Sublease structure: Mason ↔ Quikserve ULC (Master Lease); Marcus is sublessee under 2010 Sublease assigned to him 2019. No privity to landlord.
- Plaintiffs amended out Quikserve April 22, severing the only contractual chain.
- L.M.L. Group Inc. has paid water directly to City of Toronto: $4,487.05 verified bank trail.

Recent strategic decisions (full log in _matter-decisions.md):
- 2026-04-22 — Declined Quikserve co-rep agreement. Reason: indemnity creates conflict.
- 2026-04-25 — Lead with no-privity defence, file Form 9A.

Court deadlines: Defence due May 3 (6 days)

Brief refreshed from email pull. Material updates merged in:
- Apr 26, 8:43am — Stephen Marsh (Quikserve counsel) sent updated Master Lease scan
- Apr 27, 9:12am — Marcus confirmed no other bank account holds water payments

Ready to go. What are we working on?
```

End every orientation with: **"Ready to go. What are we working on?"**

### Step 4 — Do the Work (and Save As You Go)

Proceed with whatever the user needs.

**CRITICAL: Every substantive task has three parts — (1) do the work, (2) save the relevant matter file(s), (3) update the tracker. A task is not complete until all three land in the same response. Sessions end without warning. There is no "later."**

#### Three-File Architecture

| File | Shape | Length | Lifecycle |
|------|-------|--------|-----------|
| `_matter-brief.md` | Current-state snapshot + demoted historical | No cap | Live sections rewritten as facts change; resolved items demoted to bottom. |
| `_matter-decisions.md` | Strategic decisions + reasoning | None | Append-only. Never edit, reorder, or remove. |
| `_matter-comms.md` | File-specific operational rules | None | Append-only. Never edit, reorder, or remove. |

Misplacing content between files is the single failure mode that destroys institutional memory across sessions. Reasoning goes in the decisions log, not the brief.

#### Drafting Disciplines

**Source-First Drafting.** Substantive legal drafting (redlines, demand letters, opinion letters, pleadings, closing docs) starts from the source document on disk, not from the brief or memory. Briefs orient; they do not authorize. When redlining a counterparty's draft, build the baseline from the counterparty's file, not a summary. Dates, dollars, section numbers, party names, addresses, quoted text — all come from a fresh read of the source. If the source isn't on disk, stop and request it. Save it to the matter folder. Then draft.

**Citation Discipline.** Before any of these appear in client-facing output (emails, letters, opinions, redlines, advice memos, tracker Timeline entries):

- Section numbers and clause references
- Dollar figures and dates
- Party names, entity numbers, property addresses
- Quoted or paraphrased clause text

…open the source and confirm. Do this even when confident — confidence is often what produces the error. If the source isn't available, don't cite from memory. Flag the gap and either request the document or frame the advice without the citation.

**Prior-Matter Fact Discipline.** Before any categorical statement about the firm's prior involvement (or non-involvement) with a person — "you've never been retained by this person", "we never sent a letter for them" — check ALL THREE:

1. **Tracker** — Open AND Closed Matters, columns B (Client), C (Description), H (Opposing), U (Other Parties).
2. **File system** — list Open Files directory and grep folder names. A folder existing means a file existed even if the tracker doesn't reflect it.
3. **Gmail** — search for the person's email and full name. Old retainers leave email trails.

All three empty → the categorical assertion is safe. Any hit → describe what was found instead.

**Pre-Send Sourcing Check.** Any client-facing output (anything leaving the firm — letters, redlines, pleadings, demand letters, emails with substantive advice to anyone other than the lawyer) requires this inline table before sending:

| Claim | Source | Confidence |
|-------|--------|------------|
| Purchase price $140,000 | APS Form 502, Dec 5 2025, s.1 | verified |
| Closing date Apr 21 2026 | Amendment Form 570, Feb 13 2026 | verified |
| Hunter's director status | [TBC — corporate profile stale since Feb 2024] | unverified |

One row per factual claim (dates, dollars, cites, party names, addresses, roles, quoted text). Generic legal reasoning and statutory cites don't need rows. Rows marked "inferred" or "unverified" block the send until the lawyer confirms in writing, resolves to a verified source, or rewrites the output.

**Instruction Ledger for Substantive Drafts.** When producing substantive drafts (redlines, pleadings, opinion letters, closing docs), maintain an inline ledger before the draft lands:

| Provision | Instruction source | Category |
|-----------|--------------------|----------|
| s.2.2 price $0 upfront | Client email Apr 18 2026, 4:18 PM | instructed |
| s.2.6 acceleration remedy | [lawyer-side addition for enforceability] | discretionary |
| s.5.4 sanctions rep | [lawyer-side professional-obligation item] | discretionary |

"Discretionary" items (substantive additions the client didn't ask for) get separate sign-off from the lawyer before the draft goes out. Purpose: make them visible so the lawyer can decide what to include where the client said "no negotiation" or buyer friction is a risk.

**Privilege Screen.** Before any outgoing communication to anyone other than the client, compare the draft against the brief's "Positions Taken / Advice Given" and "Risks & Issues" sections. Flag phrasings that paraphrase internal material. Examples that should flag:

- Draft to buyer reads "my client is prepared to accept" when the internal walk-away is meaningfully higher.
- Draft to opposing counsel reads "we are concerned about X" where X is a flagged internal weakness.
- Draft to counterparty reads "client accepts the risk of Y" where Y came from a written client instruction.
- Draft paraphrases the lawyer's own advice ("my lawyer thinks…").

Output: a short list inline before the send, one line per flagged phrase with the matching brief entry. the lawyer approves or rewords. Surface, don't auto-block.

#### When to Save

**`_matter-brief.md` save triggers** (current state changed):
- Reviewed a document and formed conclusions affecting Risks / Open Items / Positions
- Drafted something material (letter, clause, memo, pleading)
- New role identified or role changed
- Status, stage, or summary changed
- User says "save that," "update the brief," etc.

**`_matter-decisions.md` append triggers** (strategic call made):
- Declined or accepted a counterparty's term where reasoning matters
- Set or changed a settlement floor / ceiling
- Took a strategic position the user agreed to (forum, pleading theory, scope)
- Made a fee or scope decision
- Declined or limited representation
- User says "log that decision"

**`_matter-comms.md` append triggers** (operational rule set):
- User states a preference for how this matter should be handled going forward
- Client states a preference about how to be communicated with that should bind future sessions

**Tracker save triggers** (always when any above fires): Last Activity → today; Timeline → append one-line entry; Next Action → update if changed.

**Do NOT save** after quick factual lookups or purely conversational turns with no substantive output.

A strategic decision typically produces TWO saves: brief update (current state) AND decisions log append (the reasoning).

#### Brief Format (`_matter-brief.md`)

Current-state snapshot. Tracker timeline holds historical record. Decisions log holds strategic reasoning. Brief holds only what's live now.

**No length cap. Demote, don't prune.** When an item in Risks & Issues, Positions Taken, Open Items, or Key Terms becomes resolved or superseded:

1. Move it to `## Resolved / Historical` at the bottom (create the section on first demotion).
2. Append a one-line resolution note. Example: `- 2026-04-22 — Quikserve co-rep agreement: declined; full reasoning in _matter-decisions.md.`
3. Leave the original wording intact — don't rewrite history, annotate it.

**Soft warning at 250 lines.** If a save would push the brief past 250 lines, surface it before writing: "Brief is at [N] lines. Want me to refactor (demote superseded items, move reasoning to _matter-decisions.md) or save as-is?" Wait for direction; never silently prune. Same threshold as the matter-tracker skill, so briefs from either skill are sized consistently.

**Sweep for demotion candidates on every brief refresh, before merging new content.** New email content often resolves an open item or supersedes a flagged risk — demote in the same edit.

**Reasoning still belongs in `_matter-decisions.md`.** If a brief Risk or Position runs more than three or four lines because it includes reasoning, split it: short summary stays in the brief with "full reasoning in _matter-decisions.md"; full reasoning goes in the decisions log entry of the same date.

**Format:**

```markdown
> PRIVILEGED & CONFIDENTIAL — Solicitor-Client Privilege / Work Product

# [Client Name] — [File #]

## Matter Summary
[2-3 sentences: what this is, who the parties are, what stage]

## Roles
- Name (role) — source: [email date / doc filename + page / tracker col X]

## Key Terms / Provisions
[Transactional only — price, term, material conditions, unusual clauses]

## Risks & Issues Flagged
- [Concise bullets]

## Positions Taken / Advice Given
- [Current state, not historical reasoning]

## Open Items
- [Unresolved, pending, or needs follow-up]

## Tracked Threads
- [Gmail thread ID] — "[short subject label]" — last seen YYYY-MM-DD

## Resolved / Historical
- [YYYY-MM-DD — Demoted item with one-line resolution note. Created on first demotion; append-only.]

## Last Updated
[Date]
```

Omit sections that don't apply, except the **Roles block, which is mandatory** — the one place each person is pinned to a source. Paraphrasing a role in an outgoing email without confirming it here is how role errors leak into client-facing work.

**Tracked Threads block — what it's for.** The persistence layer for Pass C. Every Gmail thread ever identified as belonging to this matter gets one line: thread ID, short subject label, date of most recent processed message. Pass C iterates this list via `get_thread` on each load. The list grows; it doesn't shrink. Dormant threads STAY in the block — never demote or delete them. Pass C only iterates `## Tracked Threads`, so a demoted thread ID catches nothing: a future reply on it from an unknown sender with no matter keywords would be missed by every pass. The cost of keeping a dormant thread is one cheap `get_thread` per load; the cost of dropping it is a silently missed reply.

**Source tagging in the body:**

- Unmarked → read directly from a source document on file
- `[inferred]` → derived from other facts
- `[per client, unverified]` → stated by client but not document-backed
- `[TBC]` → known to need a source

Use sparingly but honestly. An untagged claim is a guarantee that it came from a source you actually read.

**Privilege header**: Always include at the top. The brief stays internal — never shared with clients, opposing parties, or production. Same for decisions log and comms file.

#### Decisions Log Format (`_matter-decisions.md`)

The file's strategic memory. Capture decisions whose REASONING you'd want a future session to know.

**Append-only. No cap.** Never edit, reorder, or remove. If a decision is reversed, append the reversal as a new dated entry.

```markdown
> PRIVILEGED & CONFIDENTIAL — Solicitor-Client Privilege / Work Product

# [Client Name] — [File #] — Decisions Log

- 2026-04-22 — Declined Quikserve co-rep agreement. Reason: indemnity creates direct conflict between sublandlord and subtenant.
- 2026-04-25 — Recommended $225/hr full / half during training (vs. Iullia's $250/hr ask). Reason: bridges her hospitalist anchor without conceding the partnership-stage frame.
```

Routine document review and email drafting don't warrant entries — those go in the tracker timeline.

#### Comms / Client Preferences Format (`_matter-comms.md`)

File-specific operational rules. Loaded at the top of every session, treated as binding.

**Append-only. No cap.** Same rules as decisions log.

```markdown
> PRIVILEGED & CONFIDENTIAL — Solicitor-Client Privilege / Work Product

# [Client Name] — [File #] — Communications & Client Preferences

- 2026-04-27 — Written only on this file; no calls. Per the lawyer.
- 2026-04-15 — Always cc Laura Kim on opposing counsel correspondence. Per the lawyer.
```

Only entries that bind future sessions belong here. One-off instructions don't.

#### Universal Save Procedure

Applies to all three matter files. Save only files whose content actually changed this turn.

For each file:

1. **Backup before write.** Copy existing file (if any) to `backups/` with date inserted before `.md`. Examples: `_matter-brief.md` → `backups/_matter-brief.2026-04-27.md`. Create `backups/` if missing. One backup per file per day; same-day overwrites fine. Never auto-delete older backups.
2. **Concurrent-session check.** Compare current mtime against the Step 2 mtime. If on-disk is later, another session edited the file. Do NOT silently overwrite. Tell the user: "[filename] was modified by another session at [time]. Re-read and merge before saving?" Wait. If no mtime was recorded at Step 2 (file didn't exist then but exists now, etc.), read current mtime as baseline now.
3. **Write.** Brief: merge into the snapshot — rewrite live sections, demote superseded items to `## Resolved / Historical`. Decisions / comms: append at the bottom; never edit existing entries.
4. **Verify.** Re-open and confirm new content is present. If verification fails, alert the user and point to the most recent backup.

**If no file exists**, create from the format spec. Don't create empty files preemptively.

**If matter folder path is unresolved** (Step 2 failed), save to the tracker's directory as `<filename>-[client-name].md` and tell the user to move it.

#### Tracker Update (lightweight inline write)

Three targeted cell updates on the matter's row:

1. **Last Activity (column G)**: today's date.
2. **Timeline (column J)**: append `YYYY-MM-DD -- [brief description]`. Never overwrite prior entries.
3. **Next Action (column I)**: update only if changed.

**Lock check first.** If `~$matter-tracker.xlsx` exists beside the tracker, it is likely open in Excel — warn the user and wait before writing (same rule as the matter-tracker skill; a write against an open workbook can fail or corrupt it). **Then backup.** Copy `matter-tracker.xlsx` to `backups/matter-tracker-backup-YYYY-MM-DD.xlsx`. After write, re-open with openpyxl to confirm clean load. Never auto-delete older backups.

Use xlsx skill's openpyxl approach. Keep the row reference from Step 1.

**If tracker write fails** (permissions, file locked), don't block work. Flag once ("Couldn't update the tracker — may be open elsewhere") and continue.

#### Calendar Sync Hook

After the tracker write:

**Case 1 — Next Action (column I) changed to a new dated entry.** Call `calendar-sync.upsert_deadline` with `category="FUP"`, `slug="nextaction"`, the new date, and description. If now undated or empty, call `calendar-sync.cancel_deadline` with the same key.

**Case 2 — A dated third-party follow-up surfaced** (e.g., "Email Nina Bauer on April 22 if no defence filed"). Call `calendar-sync.upsert_deadline` with `category="TFUP"`, descriptive slug, date, description.

**Good TFUP signal:** concrete date AND concrete action against a third party. "Check in sometime" is not a TFUP; "Email her April 22 if no defence" is. When in doubt, ask the lawyer before calendaring.

**Resolving items:** If the user closes an item ("done, sent the email"), call `calendar-sync.cancel_deadline`. Clean up expired FUP/TFUP on the next inline update if today > event date.

**Tell the user** briefly when a calendar change landed: "Calendar updated: follow-up on Apr 22." Silent changes erode trust.

If calendar-sync or Calendar MCP unavailable, skip and note once.

#### What to Tell the User After Saving

After the **first** save in a session, name what was saved: "Saved: brief, decisions, tracker." Subsequent saves stay silent unless something failed or the user asks.

## Important Rules

1. **Save inline, never later.** Task is not complete until matter file(s) AND tracker are updated in the same response.
2. **Step 2.5 always runs.** Three passes (C → A → B), past 7 days minimum, 30 days max. Read mode scales with brief freshness; the search itself doesn't. Update `## Tracked Threads` on every refresh or Pass C silently regresses.
3. **Three files, three lifecycles.** Brief = snapshot, demote don't delete. Decisions and comms = append-only, never edited.
4. **Source-first for everything that leaves the firm, AND for any categorical claim about prior firm involvement.** Tracker + filesystem + Gmail check before any "we never" assertion.
5. **Backup, mtime-check, write, verify — every save.** Same for the tracker.
6. **Calendar sync after every tracker change.** If calendar-sync errors, log once and continue.
