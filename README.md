# Lawyered Matter Tracker

A practice management system for solo and small law firms, powered by [Claude](https://claude.ai).

- You have client folders and an Excel spreadsheet — that's your whole CRM
- Instead of clicking through software, you talk to Claude to manage your files
- **"New matter Smith"** — searches your Gmail and client folders, builds a timeline, runs a conflict check, and adds the matter to your tracker
- **"Daily triage"** — scans your inbox, matches emails to open matters, flags court deadlines, and tells you what needs attention
- **"Let's work on Smith"** — loads the full context for a matter so you can pick up mid-conversation where the last session left off
- **"Update matter Smith"** — pulls new emails and documents since the last update, merges them into the timeline
- Everything stays local — no cloud database, no SaaS subscription, no vendor lock-in
- Includes an optional web dashboard for visual tracking

![Dashboard](webapp/static/img/screenshot-dashboard.png)

## Prerequisites

- [Claude Desktop](https://claude.ai/download) or [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- **Gmail MCP server** connected — the skills pull emails to build timelines and run triage. Without it, the skills still work but fall back to folder-only scanning.
- **Google Calendar MCP server** connected *(optional, for calendar-sync)* — pushes limitation dates, court deadlines, and follow-ups to a dedicated "Key Dates" Google Calendar. Skill falls back gracefully if not connected.

## Setup

### 1. Install the skills

Install the five skills from this repo through Claude's UI, or import them from the `skills/` directory.

> **Want Clio integration?** There's a [Clio-integrated variant of this repo](https://github.com/lawyered0/lawyered-matter-tracker-clio) that adds one-way sync from the tracker to [Clio Manage](https://www.clio.com/ca/clio-manage/) via the [clio-mcp](https://github.com/lawyered0/clio-mcp) server. Use that one instead if you want matters to appear in Clio automatically.

### 2. Set up your client directory

Your client directory should look like this — each client gets a subfolder:

```
My Client Files/
├── CLAUDE.md              ← copy from this repo, edit for your jurisdiction
├── matter-tracker.xlsx    ← the tracker (created automatically on first use)
├── Smith, John/           ← client folder
├── Rivera Holdings/       ← client folder
└── ...
```

Copy `CLAUDE.md` from this repo into your client files directory and edit it to fill in your firm name, initials, court email domains, and limitation statutes for your jurisdiction.

### 3. Start working

Open Claude Desktop (or run `claude` from the CLI) with your client files directory as the working directory. Then just say things like:

- "Run the daily triage"
- "Open a new matter for Smith v Jones"
- "Let's work on the Garcia file"
- "Update the timeline for matter 2026-003"
- "Run a conflict check for Acme Corp"

The tracker spreadsheet is created automatically the first time you open a new matter.

## How It Works

Five Claude skills handle the core workflows:

| Skill | What It Does |
|-------|-------------|
| `daily-triage` | Scans Gmail for new emails, matches them to open matters, surfaces urgent items, auto-fills missing tracker fields, and presents a prioritised triage summary |
| `matter-tracker` | Opens, updates, and closes matters by pulling from Gmail + client folder files to build timelines. Runs conflict checks, tracks limitation periods, manages calendar sync, and maintains the Excel tracker |
| `work-on-matter` | Loads context for an existing matter at the start of a work session — reads the tracker row plus a current-state brief file (`_matter-brief.md`) from the matter folder, then orients you. As you do substantive work, saves an updated brief and writes inline tracker updates (Last Activity, Timeline, Next Action) so the next session picks up where you left off. Includes source-first drafting, pre-send sourcing checks, an instruction ledger for substantive drafts, and a privilege screen on outbound communications |
| `calendar-sync` *(helper)* | Pushes limitation dates, court deadlines, and follow-ups to a dedicated "Key Dates" Google Calendar. Invoked internally by `matter-tracker` and `work-on-matter` — not user-facing. Requires a Google Calendar MCP server |
| `overdue-triage` | The periodic deep sweep. Reviews every open matter for past-date items across Next Action (col I), Limitation Deadline (col R), and Court Deadlines (col S); investigates each, confirms with you one item at a time, and applies approved changes in a single batched write. Meant to run every few weeks |

### Daily Triage
Searches Gmail for recent emails, matches them against open matters by name/email/keyword, classifies urgency (court emails are always urgent), and presents a scannable summary. Auto-fills missing contact info when confident. Categorises unmatched emails into: active matters not yet tracked, new client inquiries, leads, and non-legal.

### Matter Tracker
The CRM engine. "New matter Smith" triggers a full Gmail search + folder scan, builds a chronological timeline, runs a conflict check, and adds the matter to the spreadsheet. "Update matter Smith" pulls new activity since the last update and merges it in. "Close matter Smith" finalises and moves the row to the Closed Matters sheet. Always confirms before writing.

### Work on Matter
"Let's work on Smith" reads the tracker row and the matter brief (`_matter-brief.md`) from the matter folder, then summarises where things stand so you can pick up immediately. As you do substantive work (review documents, draft letters, give advice), the skill saves an updated brief and writes inline tracker updates (Last Activity, Timeline append, Next Action) in the same response — there's no separate "save" step, because sessions can end without warning. Includes source-first drafting guardrails (every dollar figure, section reference, and party name gets confirmed against the source document before it lands in client-facing output), a pre-send sourcing check that surfaces unverified claims before any outbound communication leaves the firm, an instruction ledger that ties every provision in a substantive draft to either a client instruction or a flagged discretionary addition, and a privilege screen that catches internal reasoning bleeding into outbound comms.

### Calendar Sync *(helper, not user-facing)*
Projects the tracker onto a dedicated Google Calendar. Court deadlines, limitation dates, follow-ups, and third-party pings each get a colour-coded event with a 14/7/2/0-day reminder schedule. The tracker is the source of truth; the calendar is a read-only projection.

### Overdue Triage
The monthly (or quarterly) sweep. Walks every open matter, finds past-date items in the deadline columns, searches Gmail and the matter folder to figure out which ones were actually dealt with, and confirms each one with you before a single batched write. Items that look genuinely unresolved get surfaced as a red-flag list with suggested next actions.

## Spreadsheet Schema

The tracker uses two sheets ("Open Matters" and "Closed Matters") with these columns:

| Col | Header | Purpose |
|-----|--------|---------|
| A | File # | Auto-assigned `YYYY-NNN` |
| B | Client Name | Format: `Entity (Principal)` |
| C | Matter Description | Brief description |
| D | Status | Open / Closed |
| E | Date Opened | YYYY-MM-DD |
| F | Date Closed | Blank until closed |
| G | Last Activity | Updated on every interaction |
| H | Opposing Party | If applicable |
| I | Next Action / Deadline | Key upcoming step |
| J | Timeline | Chronological log, one line per event |
| K | Client ID Verified | Checkmark or Pending |
| L | Conflict Check Done | Checkmark or Pending |
| M-O | Client Email/Phone/Address | Contact info |
| P | Discovery Date | For limitation tracking |
| Q | Limitation Statute | Dropdown of configured statutes |
| R | Limitation Deadline | Auto-calculated or manual |
| S | Court Deadlines | JSON array of bespoke deadlines |
| T | Matter Folder | Subfolder name for client files |
| U | Other Parties | For conflict check coverage |
| V | Matter Type | Free-text classification (Litigation / Solicitor / Transactional / etc.) |

## Optional: Web Dashboard

The `webapp/` directory contains a Flask web dashboard that reads and writes the same `matter-tracker.xlsx` file. Copy it into your client directory and run:

```bash
pip install flask openpyxl
python app.py
```

Open [http://localhost:5001](http://localhost:5001). This is a supplementary visual tool — the primary workflow is conversational through Claude.

## Author

[@bitgrateful](https://x.com/bitgrateful)

## License

MIT
