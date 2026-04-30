---
name: work-on-matter
description: "Use this skill whenever the user wants to resume work on an existing matter or client file. Trigger on phrases like 'let's work on matter [name]', 'let's work on [name]', 'pull up [name]', 'open the [name] file', 'where are we with [name]', 'bring yourself up to speed on [name]', or any request to pick up, continue, or revisit a client matter. Also trigger on 'what do we have on [name]' or 'refresh yourself on [name]'. Do NOT trigger on 'new matter', 'update matter', or 'close matter' — those belong to the matter-tracker skill. This skill loads context from the tracker plus three per-matter files (`_matter-brief.md`, `_matter-decisions.md`, `_matter-comms.md`), ALWAYS runs a bounded Gmail pull on the past 7 days at the start of every session and refreshes the brief with any material findings before orienting, and does inline saves as work progresses."
---

# Work on Matter — Context Loader

## Purpose

This skill loads context for an existing matter so you can pick up where you left off in a new chat session. It reads from the matter tracker spreadsheet plus up to three per-matter files in the client's matter folder:

- `_matter-brief.md` — current-state snapshot
- `_matter-decisions.md` — append-only log of strategic decisions and reasoning
- `_matter-comms.md` — append-only log of file-specific communications and client preferences

**On every load, the skill does a bounded Gmail pull on the past 7 days and merges any material findings into the brief BEFORE the orientation lands.** This is not optional and is not gated on staleness — orienting from a brief that's even a few days old will silently miss everything that came in since, and that defeats the point of context loading. As you do substantive work, the skill saves to whichever file the new content belongs in and writes a lightweight tracker update (Last Activity, Timeline, Next Action), so the file system stays current without a separate "update matter" step.

## When This Runs

The user says something like "let's work on matter Nguyen" or "pull up the Rivera file." They want you oriented and ready to answer questions or do work on that matter.

## Conventions

- **"the lawyer"** refers to the user.
- **"Client"** refers to the person/entity who retained the lawyer on the matter.

## Dependencies

- **Matter tracker spreadsheet**: `matter-tracker.xlsx` lives in the Open Files directory alongside the matter folders. To find it: check the current working directory first, then the parent directory, then one level up. If not found after three checks, ask the user. Do not glob recursively from the home directory.
- **Filesystem access**: The Open Files directory contains both the tracker and all matter folders as sibling subdirectories.
- **calendar-sync skill**: Invoked after the inline tracker write when the Next Action changed to a new dated value or when the work surfaced a concrete dated third-party follow-up. See "Calendar Sync Hook" in Step 4. If unavailable, proceed without — never block the tracker write.
- **Gmail MCP tools**: `gmail_search_messages`, `gmail_read_thread`, `gmail_read_message`. Required for the Step 2.5 Email Refresh, which runs on every load. If unavailable, surface the gap loudly in the Step 3 orientation (see Step 2.5 fallback language) — do not proceed silently as if the brief is fresh.

## Workflow

### Step 1 — Find the Matter in the Tracker

1. Extract the client/matter name from the user's message.
2. Load the matter tracker spreadsheet. Check CWD, then CWD's parent, then one level up. If not found after three checks, ask the user for the path. Search the "Open Matters" sheet for a matching row (case-insensitive partial match on Client Name, Matter Description, or Opposing Party). Also check "Closed Matters" if no match found on Open.
3. **If multiple matches, present a numbered chooser before doing anything else.** Names routinely match multiple open matters, and a vague "which one?" forces the user to remember file numbers they shouldn't have to. Format the chooser like this:
   ```
   Multiple matches for "[name]". Which one?
   1. 2026-XXX | [Client Name] — [Matter Description] (Last Activity [date])
   2. 2026-YYY | [Client Name] — [Matter Description] (Last Activity [date])
   3. 2026-ZZZ | [Client Name] — [Matter Description] (Last Activity [date]) [CLOSED]
   ```
   Mark closed matters with [CLOSED]. Sort open matters first, then closed. Wait for a numeric answer (or a clarifying name) before proceeding.
4. If no match found on either sheet, tell the user there's no existing file for that name. Ask whether they want to open a new matter (which will trigger the matter-tracker skill's "new matter" workflow) or if they may have the name wrong. If the name is close to several existing matters, suggest the closest matches.

### Step 2 — Resolve the Matter Folder (and Subfolder)

Client folders often contain subfolders for separate matters (e.g. "Real Estate Purchase/", "Small Claims - Damage Deposit/", "Incorporation/"). This step resolves the correct folder so the matter files are read from and saved to the right place.

1. From the matching row, get the Matter Folder name (column T).
2. **If column T has a value**: the matter folder is a sibling directory of the tracker file. List the subdirectories alongside `matter-tracker.xlsx` and find the one matching column T. This is the **client folder**: `<open-files-dir>/<matter-folder>/`.
3. **If column T is blank**: list all sibling directories of the tracker file and do a case-insensitive fuzzy match against the client name — try last name, first name, full name, entity name, and permutations. If a match is found, use it and note that the tracker's Matter Folder column should be updated. If no match, proceed without the folder.
4. **Check for matter-specific subfolders.** List the immediate contents of the client folder. If subdirectories exist:
   - Check whether any subfolder name matches the matter description (column C) or opposing party (column H) using case-insensitive keyword matching.
   - If a matching subfolder is found, that subfolder is the **resolved matter folder**.
   - If no subfolder matches but `_matter-brief.md` exists at the client folder's top level, use the client folder.
   - If the client folder has subfolders, none match, and no brief exists at the top level either, ask the user which subfolder this matter lives in.
   - If the client folder has no subdirectories, the client folder itself is the resolved matter folder.
5. Look for the **three matter files** in the resolved matter folder:
   - `_matter-brief.md` — current-state snapshot
   - `_matter-decisions.md` — append-only decisions log
   - `_matter-comms.md` — append-only communications & client preferences
6. Read each file that exists. **Record each file's mtime when you first read it** — you'll need it in Step 4 for the concurrent-session safety check.
7. If a file doesn't exist, that's fine — files are created on first need. Don't create empty files preemptively.
8. If you can't find the client folder at all, proceed without any of the three files — you can still orient from the tracker data alone. Flag that the matter files can't be read or written until the folder is identified.

**Remember the resolved matter folder path and the mtimes of the files you read** — both are needed in Step 4.

### Step 2.5 — Email Refresh (ALWAYS run; updates the brief before orientation)

The brief is only as current as the last time someone refreshed it. Orienting from a brief that's even a few days old will silently miss anything that came in since, and the user will end up pasting emails into the chat to fill the gap. The previous version of this step was conditional on the brief being "stale" by some 7-day rule, which routinely produced exactly that failure mode (brief touched 3 days ago, five new emails in the meantime, freshness pull never fires, stale orientation lands). That conditional gating is gone.

**This step ALWAYS runs on load.** Every time. No conditionals. No "skip if recent." The cost is one Gmail search; the cost of skipping it is a confidently wrong orientation.

This is NOT the matter-tracker skill's full update workflow — no folder scan, no full timeline rebuild, no Gmail tracker writes. It is a targeted Gmail pull on a fixed lookback window, followed by a brief refresh if anything material was found, so the orientation in Step 3 reflects reality.

**Lookback window:**

- **Default: past 7 days.** Always pull at least the past 7 days regardless of how recent Last Activity or the brief's Last Updated date is.
- **If Last Activity (column G) or the brief's `## Last Updated` is older than 7 days**, extend the lookback to that older date minus 1 day for buffer.
- **Cap at 30 days.** If the brief is genuinely months stale, tell the user "Brief is very stale — recommend running 'update matter [name]' for a full refresh" and pull the last 30 days here regardless.

**How to run:**

1. Build the Gmail query from the matter row: client name (entity AND principal name in brackets, separately) joined with OR; opposing party name if column H is populated; any matter-specific keywords from column C that are unusual enough to be useful (e.g., a property address, a court file number). Don't pad with generic terms like "lease" or "claim" — they over-match.
2. Use `gmail_search_messages` with `newer_than:` set to the lookback window from above.
3. Read each thread in full with `gmail_read_thread`. Snippets miss dates, deadlines, dollar figures, and names. There is no point doing this pull if you read snippets.
4. For each thread, extract: sender, date/time, what changed (instructions given, documents exchanged, deadlines set, scheduling, court correspondence). Court emails (`@court.example.gov`, `@superior-court.example.gov`, any tribunal domain) are always reported even if they look administrative.

**Brief refresh (the part that fixes the stale-brief failure mode):**

After the pull, classify each finding:

- **Material** — affects current state: new role identified, role changed, new risk surfaced, advice given/received, position taken, deadline set or moved, document exchanged that bears on the matter, status/stage change.
- **Informational** — scheduling chitchat, "got it thanks" replies, calendar invites, anything that does not change what's true about the matter right now.

If ANY findings are material, refresh the brief BEFORE proceeding to Step 3:

1. Merge each material finding into the appropriate brief section: new roles into Roles (with email date as source citation), new flags into Risks & Issues, new advice/positions into Positions Taken, new open items into Open Items, transactional changes into Key Terms.
2. Update `## Last Updated` to today's date.
3. Save using the **Universal Save Procedure** in Step 4 (backup, mtime check, write, verify). If no brief existed before, create it from scratch.
4. Also update the Last Activity (column G) and append a Timeline entry (column J) on the tracker row using the lightweight tracker write described in Step 4. Use a single combined Timeline line summarizing the email pull (e.g. `2026-04-28 -- Brief refreshed from email: opposing counsel sent updated lease scan; client confirmed bank trail.`).
5. Then continue to Step 3 and orient from the now-current brief.

If all findings are informational (or there were none), do NOT touch the brief or tracker. Step 3 will still surface the informational items in the "What's new" block so the user sees them, but no save fires.

**What NOT to do:**
- Do not skip this step because the brief "looks recent." Recent is not current.
- Do not do a folder scan. That belongs to matter-tracker's update workflow.
- Do not rewrite history in the brief — only refresh sections that changed. Decisions log and comms file are untouched here.
- Do not exceed the 30-day lookback cap.

**If Gmail tools are unavailable:** Skip the pull and note it in the Step 3 orientation: "Couldn't check Gmail — orienting from brief alone, which may be stale. Recommend running 'update matter [name]' if anything important might have come in." Do NOT proceed silently.

### Step 3 — Orient and Summarize

Present a concise orientation. The order matters: comms / preferences first (rules of engagement should be visible before drafting starts), header and brief in the middle, deadlines and "what's new" last (freshest in mind when work begins).

Use this skeleton, omitting blocks that don't apply:

1. **Header**: file number, client, description, status, last activity, next action.
2. **Comms / Preferences block** (if `_matter-comms.md` has any entries) — print verbatim, treat as binding for this session.
3. **Brief content** — extract the live story from the (now-refreshed) `_matter-brief.md`. Step 2.5 has already merged any material email findings into it before this point, so you are reading from a current brief, not yesterday's snapshot. If no brief exists, say so and quote the tracker timeline if there is one.
4. **Recent decisions** (if `_matter-decisions.md` exists) — show the last 5 entries verbatim with a note that the full log is in the file.
5. **Deadline alerts** — limitation deadline within 6 months, court deadlines within 60 days.
6. **What's new from the email pull** (always include this block) — one line per email/thread from Step 2.5's pull, prioritized by urgency, with [URGENT] tag on court emails or items with deadlines under 7 days. If Step 2.5 found material items, lead with "Brief refreshed from email pull. Material updates merged in:" and list them. If Step 2.5 found only informational items, list them under "Informational only (brief unchanged):". If nothing came in: "No new email activity in the past [N] days." If Gmail was unavailable: "Couldn't check Gmail — orienting from brief alone, which may be stale. Recommend running 'update matter [name]' if anything important might have come in."

**Example orientation (rich case — all blocks present):**

```
Matter: 2026-148 | Michael Torres
Description: Small Claims defence (utility dispute)
Status: Open
Last Activity: 2026-04-25
Next Action: 2026-05-03: Defence due

Comms / Preferences on file (treat as binding):
- 2026-04-23 — Written only on this file; no calls. Per the lawyer.

From the matter brief:
- Sublease structure: Pacific Holdings (Master Lease); Torres is sublessee under 2010 Sublease assigned to him 2019. No privity to landlord.
- Plaintiffs amended out National Food Corp April 22, severing the only contractual chain.
- Metro Services Inc. has paid water directly to city: $X,XXX.XX verified bank trail.

Recent strategic decisions (full log in _matter-decisions.md):
- 2026-04-22 — Declined co-defendant co-representation agreement. Reason: indemnity creates conflict between sublandlord and subtenant.
- 2026-04-25 — Lead with no-privity defence, file applicable form. Reason: Plaintiffs' own pleadings contradict their tenant theory.

Court deadlines: Defence due May 3 (6 days)

Brief refreshed from email pull. Material updates merged in:
- Apr 26, 8:43am — Stephen Mitchell (co-defendant counsel) sent updated Master Lease scan (now in Roles + Open Items)
- Apr 27, 9:12am — Michael confirmed no other bank account holds water payments (now in Risks & Issues)

Ready to go. What are we working on?
```

**Example orientation (lean case — brief only, no material email findings):**

```
Matter: 2026-088 | Robert Daniels
Description: Application to appoint arbitrator
Status: Open
Last Activity: 2026-04-26
Next Action: 2026-05-28: Hearing [court file #]

From the matter brief:
- Application is procedural; the arbitrator has confirmed appointment.
- Robert ran a parallel insurer-pressure campaign without telling the lawyer; the insurer responded with cease and desist.
- Scope clarified to Robert: arbitration / coverage / regulatory all outside this retainer.

No new email activity in the past 7 days.

Ready to go. What are we working on?
```

End every orientation with: **"Ready to go. What are we working on?"**

### Step 4 — Do the Work (and Save As You Go)

Proceed with whatever the user needs — review documents, draft things, answer questions, etc.

**CRITICAL: Every substantive task has three parts — (1) do the work, (2) save the relevant matter file(s), (3) update the tracker. A task is not complete until all three land in the same response. Sessions crash, compact, or stop without warning. There is no "later."**

#### Three-File Architecture

Each matter folder carries up to three files, each with a distinct shape and lifecycle. Understand this before saving — it determines which file gets touched.

| File | Shape | Length | Lifecycle |
|------|-------|--------|-----------|
| `_matter-brief.md` | Current-state snapshot + demoted historical | No cap | Live sections get rewritten as facts change; resolved items move to a Resolved / Historical section at the bottom rather than being deleted. |
| `_matter-decisions.md` | Strategic decisions + reasoning | None | Append-only. Never edit, reorder, or remove existing entries. |
| `_matter-comms.md` | File-specific operational rules | None | Append-only. Never edit, reorder, or remove existing entries. |

The split exists because a snapshot and a log have different shapes. A snapshot's live sections get rewritten as facts change; a log only grows. Both files grow without caps — length on its own doesn't hurt, and an artificial cap creates clumsy refactors on long-running matters.

Misplacing content here is the single failure mode that destroys institutional memory across sessions: putting reasoning into the brief instead of the decisions log. The fix is content placement, not length policing.

#### Drafting Disciplines

These apply to every substantive piece of work, regardless of which matter file gets touched.

##### Source-First Drafting

Substantive legal drafting — redlines, demand letters, opinion letters, engagement letters, pleadings, closing documents, disclosure schedules — starts from the source document on disk, not from the brief or from memory. Briefs orient; they do not authorize.

When redlining a counterparty's draft, open the counterparty's file on disk and build the baseline from that file. Do not reconstruct the baseline from a summary in the brief, from memory of the last session, from a Gmail description of the deal, or from any derived text. The brief's job is to tell you what has already been decided and flagged; the source document's job is to tell you what the text actually says.

When drafting a letter or pleading that references dates, dollar amounts, section numbers, party names, addresses, or quoted text, each of those items comes from a fresh read of the underlying source (the APS, the lease, the endorsement, the court filing, the email), not from the brief's summary of them.

If the source document is not on disk — the counterparty sent a Google Doc link, a PDF you haven't saved, a verbal description — stop and request it before drafting. Save it to the matter folder. Then draft. Do not proceed on a paraphrase. The cost of one email to request the source is cheap against the cost of a baseline that diverges from the counterparty's actual text.

The most common failure mode this rule prevents: a draft whose section numbers, defined terms, or schedule content silently drifts away from the counterparty's text because it was rebuilt from the brief rather than read from the page.

##### Citation Discipline

Legal work lives and dies on specific citations. A section number you recall from a similar lease you never actually opened, or a date you inferred from context, is the kind of error that destroys client trust and creates real liability. Treat your memory as a prompt, not a source.

Before any of the following appear in client-facing output (emails, letters, opinions, redlines, advice memos, tracker Timeline entries):

- Section numbers and clause references (e.g. s.10.1, Article 11, Schedule B)
- Dollar figures and dates
- Party names, entity numbers, property addresses
- Quoted or paraphrased clause text

...open the source document in the matter folder and confirm the citation matches what's actually there. Do this even when you're confident. Confidence is not the signal; it's often what produces the error.

If the source isn't available in the folder, do not cite from memory. Flag the gap to the lawyer and either request the document or frame the advice without the citation. "Landlord's consent is required under the assignment provision" is always better than "landlord's consent is required under s.10.1" when you haven't actually read s.10.1.

##### Prior-Matter Fact Discipline

The same source-first principle applies to factual claims about the firm's own history with a person — whether a prior retainer existed, what was done on a closed file, whether a letter was sent on a person's behalf. Asserting "you've never been retained by this person" or "we never sent a letter for them" without checking is the same category of error as a wrong section number, with the same kind of reputational and liability risk if the assertion is wrong and lands in writing.

Before any categorical statement about the firm's prior involvement (or non-involvement) with a person or entity, check ALL THREE of these sources:

1. **Tracker** — both Open Matters AND Closed Matters sheets. Search columns B (Client Name), C (Matter Description), H (Opposing Party), and U (Other Parties).
2. **File system** — list the contents of the Open Files directory and grep for the person's name across folder names. A folder existing means a file existed even if the tracker doesn't reflect it (legacy matters from before the tracker was started often live in folders with no tracker row).
3. **Gmail** — search for the person's email address and full name. Old retainers leave email trails even when the matter folder has been archived or the tracker entry is missing.

If all three come up empty, the categorical assertion is safe. If any one of them produces a hit, describe what was found instead of asserting an absence.

##### Pre-Send Sourcing Check

Any client-facing output — defined as any document that will leave the firm (letters to clients or third parties, redlines to counterparties, pleadings, demand letters, and emails that contain substantive advice to anyone other than the lawyer) — requires a pre-send sourcing check.

Produce an inline table in chat before the output goes out:

| Claim | Source | Confidence |
|-------|--------|------------|
| Purchase price $140,000 | Purchase Agreement, Schedule A, Dec 5 2025, s.1 | verified |
| Closing date Apr 21 2026 | Amendment Agreement, Feb 13 2026 | verified |
| Wei Zhang acts for Seller | Lease Assignment signature block; Apr 2 14:04 email | verified |
| Director's status | [TBC — corporate profile stale since Feb 2024] | unverified |
| 400,000 shares transferred [Party A] → [Party B] | [inferred from 900/100 register vs 500/500 certs] | inferred |

One row per factual claim. Claims include: dates, dollar figures, section/clause cites, party names, addresses, roles, and any quoted or paraphrased text from a source document. Generic legal reasoning and statutory cites that don't depend on matter-specific facts don't need rows.

Rows marked "inferred" or "unverified" block the send. The lawyer either (a) confirms the inference in writing, (b) resolves the claim to a verified source, or (c) rewrites the output to remove or soften the claim. Do not send output with unresolved rows.

##### Instruction Ledger for Substantive Drafts

When producing substantive drafts (redlines of any length, pleadings, opinion letters, closing documents), maintain an instruction ledger alongside the source-first baseline. The ledger ties every provision in the draft to either a client instruction or a professional-obligation item.

Format (inline in chat before the draft lands):

| Provision | Instruction source | Category |
|-----------|--------------------|----------|
| s.2.2 price $0 upfront | Client email Apr 18 2026, 4:18 PM | instructed |
| s.6.1(b) buyer release deliverable | Client email Apr 15 2026 | instructed |
| s.2.6 acceleration remedy | [lawyer-side addition for enforceability] | discretionary |
| s.5.4 sanctions rep | [lawyer-side professional-obligation item] | discretionary |
| s.2.8 cash-sweep carve-out | Client Q5 answer Apr 18 2026 | instructed |

Items marked "discretionary" — substantive additions the client did not ask for — get listed separately for the lawyer's sign-off before the draft goes out. The purpose is not to forbid discretionary additions (they are often necessary and defensible) but to make them visible so the lawyer can decide what to include in a transaction where the client has said "no negotiation" or where buyer friction is a known risk.

##### Privilege Screen

Before any outgoing communication to anyone other than the client (opposing counsel, counterparty, landlord, court, adjuster, third party), compare the draft against the brief's "Positions Taken / Advice Given" and "Risks & Issues Flagged" sections. Flag phrasings that paraphrase internal material.

Examples of what should flag:

- Draft to buyer reads "my client is prepared to accept" when the brief's internal walk-away number is meaningfully higher
- Draft to opposing counsel reads "we are concerned about X" where X is a flagged internal weakness that concedes the point if disclosed
- Draft to counterparty reads "client accepts the risk of Y" where Y came from a written client instruction to proceed despite Y
- Draft paraphrases the lawyer's own advice ("my lawyer thinks the strongest argument is…")

Output: a short list inline in chat before the send, one line per flagged phrase with the matching brief entry. The lawyer approves each item or rewords. The skill does not auto-block — the lawyer may have a reason to include material — but it surfaces.

#### When to Save

Saves are routed by content type. The trigger lists tell you which file gets touched.

**`_matter-brief.md` save triggers** (current state changed):
- You reviewed a document and formed conclusions that affect Risks / Open Items / Positions
- You drafted something material (letter, clause, memo, pleading)
- A new role was identified or a role changed
- The matter status, stage, or summary changed
- The user explicitly says "save that," "update the brief," or similar

**`_matter-decisions.md` append triggers** (a strategic call was made):
- You declined or accepted a counterparty's term where the reasoning matters
- You set or changed a settlement floor / ceiling
- You took a strategic position the user agreed to (forum, pleading theory, scope)
- You made a fee or scope decision (engagement scope acceptance/decline, fee structure)
- You declined or limited representation (conflict, scope, capacity)
- The user explicitly says "log that decision" or describes a strategic call

**`_matter-comms.md` append triggers** (an operational rule was set):
- The user states a preference for how this matter should be handled going forward (channel, cc lists, tone, frequency)
- The client states a preference about how to be communicated with that should bind future sessions

**Tracker save triggers** (always when any of the above fires): Last Activity → today; Timeline → append a one-line entry; Next Action → update if changed.

**Do NOT save** after quick factual lookups (e.g. "what's the limitation deadline?"), or after a turn that was purely conversational with no substantive output.

A strategic decision typically produces TWO saves: a brief update (current state) AND a decisions log append (the reasoning). The brief tells the next session what was decided. The decisions log tells the next session why.

#### Brief Format (`_matter-brief.md`)

The brief is a current-state snapshot. The tracker timeline (column J) holds the historical record. The decisions log holds the strategic-reasoning record. The brief holds only what's live right now: who the players are, what's flagged, what's open, what's been advised.

**No length cap. Demote, don't prune.** The brief is allowed to grow as the matter grows. Length isn't what makes a brief bad. Stale items presenting as current is what makes a brief bad, and the fix for that is demotion, not deletion.

When an item in Risks & Issues, Positions Taken, Open Items, or Key Terms becomes resolved or superseded:

1. Move it to a `## Resolved / Historical` section at the bottom of the brief (create the section if it doesn't yet exist).
2. Append a one-line resolution note: how it resolved, when, and where the underlying source lives if relevant. Example: `- 2026-04-22 — Co-defendant co-rep agreement: declined; full reasoning in _matter-decisions.md.`
3. Leave the original wording intact in the demoted entry — don't rewrite history, just annotate it.

Live sections (Roles, Risks & Issues, Positions Taken, Open Items, Key Terms, Matter Summary) stay scannable because resolved items are out of the way. Institutional memory is preserved because demoted items are still in the file, just below the live sections. Future sessions reading the brief see the current state first and can scroll down for context if a question reaches back to a resolved point.

**When to demote** is a judgment call, but the bar is low: if the item is no longer something you'd act on today, it's a candidate. Settled, paid, withdrawn, completed, abandoned, superseded by a later position — all candidates. When in doubt, leave it live; demotion is reversible but a too-aggressive demotion costs nothing because the content is still in the file.

**Reasoning still belongs in `_matter-decisions.md`.** If the content pushing the brief long is the *why* behind a position rather than the position itself, that's a content-placement problem and the cure is the decisions log, not the Resolved section.

**Format:**

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
- [Key advice and positions — current state, not historical reasoning. Reasoning goes in _matter-decisions.md.]

## Open Items
- [What's still unresolved, pending, or needs follow-up]

## Resolved / Historical
- [YYYY-MM-DD — Demoted item, original wording preserved, with a one-line resolution note appended.]
- [Created on first demotion; omit until then. Append-only within this section — never delete demoted items, even old ones.]

## Last Updated
[Date of this update]
```

Omit any section that doesn't apply (e.g., skip "Key Terms" for a litigation matter, skip "Resolved / Historical" until you have something to demote), except the Roles block, which is mandatory.

**The Roles block is mandatory.** Every named party in the matter with their role and a source citation for that role. This is the single place in the brief where each person is pinned to a source. Paraphrasing a role in an outgoing email without first confirming it here is how role errors leak into client-facing work. Example:

```
## Roles
- James Patterson (Landlord principal, Metro Fashions Ltd.) — source: Lease Assignment executed Apr 7 2026, recital A
- Wei Zhang (counsel for Assignor / Seller, 9876543 Corp.) — source: signature block of Lease Assignment; confirmed Apr 2 14:04 ET email
- Rita Chen (Metro Fashions Controller) — source: Apr 17 11:39 email re security deposit wire
- Sterling Lawyers Inc. (counsel of record for Landlord) — source: s.2.8.1(c) of Lease Assignment
```

**Source tagging in the body.** Factual claims in Risks / Positions / Open Items / Key Terms follow this convention:

- Unmarked → read directly from a source document on file
- `[inferred]` → derived from other facts, not directly verified
- `[per client, unverified]` → stated by client but not backed by a document
- `[TBC]` → known to need a source

Use the tags sparingly but honestly. An untagged claim is a guarantee to the next session (and to the lawyer) that it came from a source you actually read.

**Privilege warning**: Always include the privilege header at the top. The brief stays in the firm's internal file and must not be shared with clients, opposing parties, or included in any document production. The same applies to the decisions log and comms files below.

#### Decisions Log Format (`_matter-decisions.md`)

This file is the file's strategic memory. Capture decisions whose REASONING you'd want a future session to know, not just the outcome.

**Append-only. No cap.** Never edit, reorder, or remove existing entries. If a decision is reversed, append the reversal as a new dated entry — don't rewrite the original.

**Format:**

```markdown
> PRIVILEGED & CONFIDENTIAL — Solicitor-Client Privilege / Work Product

# [Client Name] — [File #] — Decisions Log

- 2026-04-22 — Declined co-defendant co-representation agreement. Reason: indemnity clause creates direct conflict between sublandlord and subtenant; cannot act for both even with consent.
- 2026-04-25 — Recommended $XXX/hr full / half during training (vs. associate's $250/hr ask). Reason: bridges their anchor without conceding the partnership-stage frame.
- 2026-04-27 — Rejected joint retainer with co-defendant, declined call, written-only. Reason: keep paper trail clean for potential adverse positioning later.
```

Routine document review and email drafting do not warrant entries — those go in the tracker timeline. The decisions log is for the small subset of moments where a choice was made that future-you would want to know the reasoning for.

#### Comms / Client Preferences Format (`_matter-comms.md`)

This file holds file-specific operational rules: how to communicate on this matter, who to copy, what channels to use. Loaded at the top of every work-on-matter session and treated as binding for the session.

**Append-only. No cap.** Same rules as the decisions log — never edit existing entries; append reversals as new lines.

**Format:**

```markdown
> PRIVILEGED & CONFIDENTIAL — Solicitor-Client Privilege / Work Product

# [Client Name] — [File #] — Communications & Client Preferences

- 2026-04-27 — Written only on this file; no calls. Per the lawyer.
- 2026-04-15 — Always cc Taylor Kim on opposing counsel correspondence. Per the lawyer.
- 2026-03-20 — Client prefers reply by SMS for scheduling, email for substantive. Per client Apr 18 email.
```

Only entries describing rules that should bind future sessions belong here. One-off "send this without cc'ing them" instructions don't.

#### Universal Save Procedure

Applies to all three matter files. Save only the files whose content actually changed in this turn — don't touch files you didn't write to.

For each file you're about to write:

1. **Backup before write.** Copy the existing file (if any) into a `backups/` subfolder of the matter folder, with the date inserted before the `.md` extension using a period separator. Examples: `_matter-brief.md` → `backups/_matter-brief.2026-04-27.md`; `_matter-decisions.md` → `backups/_matter-decisions.2026-04-27.md`. Create `backups/` if missing. One backup per file per day; same-day overwrites fine. Never auto-delete older backups — they're the only recovery path if a save corrupts the file.
2. **Concurrent-session check.** Compare the file's current mtime on disk against the mtime you recorded in Step 2. If on-disk mtime is later, another session edited the file while you were working. Do NOT silently overwrite. Tell the user: "[filename] was modified by another session at [time]. Re-read and merge before saving?" Wait for direction. **If you didn't record an mtime at Step 2** (e.g., the file didn't exist then but exists now, or Step 2 was skipped) — read the current mtime as your baseline now. The check is then trivially safe for this write but the recorded mtime protects future writes in this session.
3. **Write.** For the brief, merge into the snapshot: rewrite live sections as facts change, and demote superseded items into the `## Resolved / Historical` section at the bottom (never silently delete them — see "Brief Format" above for the demotion rules). No length cap; let the brief grow as the matter grows. For the decisions log and comms file, append new entries at the bottom — never edit existing ones.
4. **Verify.** Re-open the file and confirm the new content is present. If verification fails, alert the user and point them to the most recent backup.

**If no file exists**, create it from scratch with the format-spec header block above. Only create files when content first warrants — don't create empty files preemptively.

**If you couldn't resolve the matter folder path** (Step 2 failed), save to the same directory as the tracker with the filename `<filename>-[client-name].md` and tell the user to move it manually.

#### Tracker Update (lightweight inline write)

This is NOT a full tracker refresh — no Gmail scan, no folder audit. Three targeted cell updates on the matter's row:

1. **Last Activity (column G):** Set to today's date.
2. **Timeline (column J):** Append a one-line entry. Format: `YYYY-MM-DD -- [brief description]`. Append with a newline; never overwrite prior entries.
3. **Next Action (column I):** Update only if what's next has changed. Otherwise leave alone.

**Before writing: back up the tracker.** Copy `matter-tracker.xlsx` to `backups/matter-tracker-backup-YYYY-MM-DD.xlsx` in a `backups/` subfolder alongside the tracker. After the write, re-open with openpyxl to confirm it loads cleanly. Never auto-delete older backups; if verification fails, point the user to the most recent backup.

Use the xlsx skill's openpyxl approach. Keep the row reference from Step 1 so you don't re-search.

**If the tracker can't be written** (permissions, file locked), don't let it block work. Flag once ("Couldn't update the tracker — file may be open elsewhere") and continue. The matter files still capture the session context.

#### Calendar Sync Hook

After the inline tracker write succeeds, keep Key Dates in step. Two cases:

**Case 1 — Next Action (column I) changed to a new dated entry.** Call `calendar-sync.upsert_deadline` with `category="FUP"`, `slug="nextaction"`, the new date, and the new description. If the Next Action is now undated or empty, call `calendar-sync.cancel_deadline` with the same key.

**Case 2 — A third-party follow-up surfaced during the work** (e.g., "Need to ping opposing counsel on April 22 if no defence is filed"). When you set a specific date AND a specific action against a third party (opposing counsel, insurer, adjuster, court clerk, expert), call `calendar-sync.upsert_deadline` with `category="TFUP"`, a descriptive slug, the date, and the description.

**Good signal for a TFUP event:** there is a concrete date AND a concrete action to take on that date. "Check in with her sometime" is not a TFUP; "Email her April 22 if no defence" is. Err on the side of restraint — if in doubt, mention the follow-up to the user and ask whether to calendar it.

**Resolving items:** If the user closes out an item ("done, sent the email"), call `calendar-sync.cancel_deadline` for that key. Expired FUP/TFUP events should be cleaned up on the next inline update if today > event date.

**Tell the user** briefly when a calendar change landed: "Calendar updated: follow-up on Apr 22." Silent changes erode trust.

If calendar-sync or the Calendar MCP is unavailable, skip these calls and note it once — the tracker/brief work is the priority.

#### What to Tell the User After Saving

After the **first** save in a session, name what was saved: "Saved: brief, decisions, tracker." (Or whichever combination applied.) Subsequent saves stay silent unless something failed or the user asks. This keeps the user informed without narrating every update.

## Important Rules

These add to or sharpen the body. Read the body for the full procedures.

1. **Save inline, never later.** A task is not complete until the matter file(s) AND the tracker are updated in the same response. Sessions end without warning.
2. **Step 2.5 always runs and refreshes the brief.** Every load pulls the past 7 days from Gmail (longer if the brief or Last Activity is older, capped at 30 days) and merges any material findings into the brief BEFORE Step 3 orients. There is no "skip if recent" — recent is not current, and the prior conditional gating routinely produced stale orientations. For a comprehensive Gmail-and-folder rebuild, the user still runs "update matter [name]" — that's matter-tracker territory. Folder scans are never done in this skill.
3. **Three files, three lifecycles.** Brief = snapshot, no length cap; live sections get rewritten as facts change, resolved items get demoted (never deleted) into a `## Resolved / Historical` section at the bottom. Decisions and comms = append-only, no cap, never edited or reordered. Misplacing content between them is the single failure mode that destroys institutional memory.
4. **Don't double-write the brief** if matter-tracker just refreshed it this session. Append-only files are unaffected — appends are always safe.
5. **Source-first for everything that leaves the firm AND for any "we never" / "you've never" claim about prior firm involvement.** Citations come from the page, not memory. Prior involvement gets a tracker + filesystem + Gmail check before any categorical assertion.
6. **Backup, mtime-check, write, verify — every matter-file save.** This is the only recovery path if a save corrupts a file or a parallel session races. Same backup discipline applies to the tracker.
7. **Calendar sync after every tracker change.** Brief work is the priority; if calendar-sync errors, log once and continue.
