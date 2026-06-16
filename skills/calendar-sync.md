---
name: calendar-sync
description: "Shared helper skill used by matter-tracker and work-on-matter to push, update, and cancel deadline events on the user's Key Dates calendar. Not a user-facing skill — it is called internally when the tracker changes. Trigger only when another skill is adding, updating, or removing a court deadline, limitation date, client follow-up, or third-party follow-up."
---

# Calendar Sync — Key Dates Helper

## Purpose

Push, update, and cancel deadline events on the Key Dates calendar so the tracker and the calendar stay in lockstep. This skill is invoked by `matter-tracker` (on NEW / UPDATE / CLOSE) and by `work-on-matter` (on inline Next Action changes). It is **not** user-facing — the user does not say "run calendar-sync."

The calendar is a projection of the tracker. Tracker is the source of truth; calendar events are derived. Nothing the user does on the calendar flows back.

## Configuration

```
KEY_DATES_CALENDAR_ID = "YOUR_CALENDAR_ID@group.calendar.google.com"
TIMEZONE = "America/Toronto"
```

If the user ever re-creates the calendar, swap the id here.

## Dependencies

- **Google Calendar MCP tools**: `list_events`, `create_event`, `update_event`, `delete_event`, `get_event`. All calls pass `calendarId=KEY_DATES_CALENDAR_ID`.
- **Read access to the tracker row** being synced — the caller passes the row data.
- If the Calendar MCP is unavailable, skip silently and tell the caller once: "Calendar MCP not connected — deadlines not pushed to Key Dates." Do not block the tracker write.

## Deadline Categories

Four categories are in scope. Every event falls into exactly one. The category determines the label in the title and the color of the event — it does NOT drive reminders (see "Reminders" below).

| Code | Label | Source | Color ID | Color name |
|------|-------|--------|----------|------------|
| `COURT` | Court | Tracker column S (JSON array of court-ordered deadlines) | 7 | Peacock (blue) |
| `LIM` | LIMITATION | Tracker column R (Limitation Deadline) | 11 | Tomato (red) |
| `FUP` | Follow-up | Tracker column I (Next Action / Deadline) when dated AND not already a court/limitation deadline | 5 | Banana (yellow) |
| `TFUP` | 3P Follow-up | Caller explicitly passes category `TFUP` for third-party prompts (opposing counsel, insurers, court clerks, experts) | 6 | Tangerine (orange) |

Color coding lets you scan Key Dates at a glance — red is existential, blue is court, yellow is your follow-up, orange is someone else's. Always set `colorId` on create_event and update_event calls.

## Reminders

Set reminders **per event** via the `create_event` / `update_event` `overrideReminders` parameter. Each reminder is `{"method": "popup", "minutes": N}` (use `popup`; add an `email` reminder on the limitation 0-day too if you want belt-and-suspenders). `overrideReminders` overrides the calendar's default notifications for that event, so each category gets the schedule it actually needs rather than one blanket default.

**Per-category reminder schedules** (minutes before the all-day event):

| Category | Days before | `minutes` values |
|----------|-------------|------------------|
| `LIM` (limitation) | 60 / 30 / 14 / 7 / 0 | 86400, 43200, 20160, 10080, 0 |
| `COURT` | 14 / 7 / 2 / 0 | 20160, 10080, 2880, 0 |
| `FUP` (follow-up) | 2 / 0 | 2880, 0 |
| `TFUP` (3P follow-up) | 2 / 0 | 2880, 0 |

Google caps reminders at 5 per event; the limitation schedule uses exactly 5.

Always pass `overrideReminders` on `create_event` and `update_event`. Limitation periods get the long 60-day runway because a missed limitation reminder is the one that ends a claim, and that early warning is the whole reason limitations go on the calendar. The tracker itself remains the primary defense against missed deadlines; these per-event reminders are the calendar-side backstop.

If the connected calendar MCP ever stops accepting `overrideReminders`, fall back to the calendar's default notification settings and tell the user once. The current MCP supports per-event reminders, so use them.

## Event Format

Every event uses the same structure.

### Title

```
[{file#} | {client_short}] {LABEL} — {short_description}
```

- `{file#}` is column A of the tracker row (e.g., `2026-070`).
- `{client_short}` is a 1–3 word slug from column B — last name if individual (e.g., "Davis"), entity short name if corporate (e.g., "Acme Group"), or the principal's last name in brackets for `Entity (Principal)` rows.
- `{LABEL}` is the category label from the table above (`Court`, `LIMITATION`, `Follow-up`, `3P Follow-up`).
- `{short_description}` is ≤ 60 chars, plain language. No legalese.

Examples:
- `[2026-070 | Davis] Court — Defence deadline (TSCC 1654)`
- `[2026-070 | Davis] LIMITATION — 2-yr Limitations Act expiry`
- `[2026-070 | Davis] Follow-up — Call Small Claims trial coordinator`
- `[2026-070 | Davis] 3P Follow-up — Ping Nina Bauer re defence`

### Time

All events are **all-day** events on the deadline date. Simple, reliable, and avoids timezone drift. Reminders are set per event via `overrideReminders` (see the Reminders section above).

### Description (event body)

The first line must be the **sync key**. The sync engine uses it to find and update events.

```
SYNC-KEY: {file#}::{CODE}::{slug}
File: {file#}
Client: {full client name from column B}
Matter: {matter description from column C}
Category: {label}
Deadline: {YYYY-MM-DD}

{full deadline description — one paragraph, no line breaks needed}

Source: {e.g., "March 12 endorsement", "Rule 1.03 — 30 days before trial", "Limitations Act, s.4", "Tracker Next Action"}
Matter folder: {column T value or "not set"}

Last synced: {ISO timestamp}
```

### Sync Key Convention

`SYNC-KEY: {file#}::{CODE}::{slug}`

- `CODE` ∈ `{COURT, LIM, FUP, TFUP}`.
- `slug` = lowercased, non-alphanumeric stripped, first 40 chars of the short_description. For `LIM` the slug is always `expiry` (only one limitation event per file). For `FUP` the slug is always `nextaction` (only one follow-up event per file for the column I value — new dated Next Actions replace the prior one).

This gives each event a stable, human-readable identity that survives edits to the title or body.

## Core Operations

### `upsert_deadline(file_number, category, date, short_description, full_description, source, client_short, client_full, matter_description, matter_folder)`

1. Compute `sync_key` from the inputs.
2. List events on Key Dates in the window `[today - 30d, date + 30d]` (covers both upcoming events and recently expired ones).
3. Find any event whose description first line equals `SYNC-KEY: {sync_key}`.
4. If found:
   - If the existing event's date matches `date` and title/body match current values, return unchanged.
   - Otherwise, `update_event` with the new title, body, date, colorId, and `overrideReminders` for this category.
5. If not found:
   - `create_event` with the title, body, date, colorId, and `overrideReminders` for this category (see the Reminders table). Pass `overrideReminders` as a list of `{"method": "popup", "minutes": N}` objects.
6. Return the event id.

### `cancel_deadline(file_number, category, slug=None)`

1. List events on Key Dates in the window `[today - 30d, today + 365d]`.
2. Find event(s) whose description first line starts `SYNC-KEY: {file#}::{CODE}` (and matches `slug` if provided).
3. `delete_event` for each match.

### `cancel_all_for_matter(file_number)`

1. List events on Key Dates in the window `[today - 30d, today + 730d]`.
2. Find every event whose title starts `[{file#} |` OR whose description first line starts `SYNC-KEY: {file#}::`.
3. `delete_event` for each.

Used when a matter is closed.

### `reconcile(matter_row)`

Full sweep for one matter. Called by `matter-tracker` after every NEW / UPDATE write, and by `work-on-matter` after an inline write.

1. Build `desired` — the set of (category, slug, date, short_description) that should exist for this row:
   - If column R (Limitation Deadline) is set and Status = Open: add `(LIM, "expiry", R_date, f"{Q_statute_label} expiry")`.
   - For each entry in column S (Court Deadlines JSON) where date > today - 1d: add `(COURT, slug_of(entry.description), entry.date, entry.description)`.
   - If column I (Next Action) is a dated entry (`YYYY-MM-DD: ...`) AND the date is not already covered by a COURT or LIM entry above AND date > today - 1d: add `(FUP, "nextaction", I_date, I_description)`.
   - Third-party follow-ups (`TFUP`) are only added via explicit `upsert_deadline` calls, not via reconcile — they come from ad-hoc work-on-matter prompts, not from tracker columns.
2. Pull `existing` — all events with title starting `[{file#} |` on Key Dates.
3. For each item in `desired`: `upsert_deadline` (create or update).
4. For each event in `existing` that is not in `desired` AND is category `COURT`, `LIM`, or `FUP` (not `TFUP`): `delete_event`. This cleans up deadlines that were pruned from the tracker (e.g., court deadline passed and was removed, Next Action changed from dated to undated).
5. `TFUP` events are left alone by reconcile — they have a separate lifecycle.

## Calling Conventions

### From matter-tracker NEW MATTER

After saving the tracker row, call `reconcile(new_row)`. Confirm to the user: "Pushed N events to Key Dates: X court, Y limitation, Z follow-up."

### From matter-tracker UPDATE MATTER

After saving, call `reconcile(updated_row)`. Report diff: "Calendar sync: 2 updated, 1 added, 1 removed."

If the update **pruned expired court deadlines** from column S, mention the deletions explicitly so the user sees them: "Removed from Key Dates: 2026-02-15 Amend claim (deadline passed)."

### From matter-tracker CLOSE MATTER

After saving, call `cancel_all_for_matter(file_number)`. Confirm: "Cancelled N events on Key Dates."

### From work-on-matter Step 4 (inline)

When the inline tracker write changes Next Action (column I) to a new dated value, call `upsert_deadline(..., category="FUP", ...)`.

When the work surfaces a third-party prompt (e.g. "follow up with Tom Boyd on April 22 if no response"), call `upsert_deadline(..., category="TFUP", slug=<derived from description>, ...)`. The caller is responsible for deciding that a TFUP event is warranted — not every mention of a person merits a calendar nudge. Good signal: there is a specific date and a specific action. Bad signal: "should probably check in with her sometime."

When the user resolves an item, call `cancel_deadline(file#, category, slug)`.

## Reconciliation Rules of Thumb

- **Never push a deadline in the past.** Skip any entry where date ≤ today.
- **One LIM event per file.** If the limitation deadline changes, `upsert_deadline` updates in place.
- **One FUP event per file.** Column I is a single next-action field; the calendar mirrors that. If the user changes Next Action from "Call coordinator" to "Serve Form 1B", the old FUP event is updated in place.
- **Multiple COURT events per file.** Each entry in column S JSON gets its own event, slugged by description.
- **TFUP events are independent of reconcile.** They only get added/cancelled via explicit calls. This prevents a tracker update from wiping ad-hoc prompts that aren't in any tracker column.
- **Closed matters have zero events.** Close always cancels everything.

## Failure Handling

- If any Calendar MCP call errors, log the error to the conversation and continue. Do not block the tracker write. Example: "Calendar sync failed for 2026-070 Davis: rate limited. The tracker is updated; run 'resync calendar' later."
- If the caller passes a malformed date (not `YYYY-MM-DD`), skip that entry and flag it to the user.
- If the caller is missing required fields (file#, category, date, short_description), ask the caller to retry — do not guess.

## Manual Resync

If events drift out of sync (MCP was down, user edited calendar directly, etc.), the user can say "resync calendar for [Name]" or "resync all calendar events". On the former, load that one row and call `reconcile`. On the latter, iterate every open matter and call `reconcile` for each. For full resync on many files, pace the calls — Google Calendar rate limits around 600 writes/minute.

## What This Skill Does NOT Do

- It does **not** read from the calendar to populate the tracker (one-way sync only).
- It does **not** fire automatically on a schedule. It only runs when another skill calls it.
- It does **not** create events on the user's personal calendar. Every event goes to `KEY_DATES_CALENDAR_ID` and nowhere else.
- It does **not** handle recurring deadlines (e.g., monthly trust replenishment checks). Those should be set up as a native recurring event by the user if wanted.
