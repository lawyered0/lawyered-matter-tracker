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

## Setup

### 1. Install the skills

Install the three skills from this repo through Claude's UI, or import them from the `skills/` directory.

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

Three Claude skills handle the core workflows:

| Skill | What It Does |
|-------|-------------|
| `daily-triage` | Scans Gmail for new emails, matches them to open matters, surfaces urgent items, auto-fills missing tracker fields, and presents a prioritised triage summary |
| `matter-tracker` | Opens, updates, and closes matters by pulling from Gmail + client folder files to build timelines. Runs conflict checks, tracks limitation periods, and maintains the Excel tracker |
| `work-on-matter` | Loads context for an existing matter at the start of a work session — reads the tracker row + a per-matter brief file so you can pick up where you left off |

### Daily Triage
Searches Gmail for recent emails, matches them against open matters by name/email/keyword, classifies urgency (court emails are always urgent), and presents a scannable summary. Auto-fills missing contact info when confident. Categorises unmatched emails into: active matters not yet tracked, new client inquiries, leads, and non-legal.

### Matter Tracker
The CRM engine. "New matter Smith" triggers a full Gmail search + folder scan, builds a timeline, runs a conflict check, and adds the matter to the spreadsheet. "Update matter Smith" pulls new activity since the last update. "Close matter Smith" finalises and archives. Always confirms before writing.

### Work on Matter
Fast context loading. "Let's work on Smith" reads the tracker row and a `_matter-brief.md` file from the matter folder. After you do substantive work (review documents, draft letters, give advice), it saves a brief so the next session can pick up instantly.

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
| J | Timeline | Chronological log with SUMMARY header |
| K | Client ID Verified | Checkmark or Pending |
| L | Conflict Check Done | Checkmark or Pending |
| M-O | Client Email/Phone/Address | Contact info |
| P | Discovery Date | For limitation tracking |
| Q | Limitation Statute | Dropdown of configured statutes |
| R | Limitation Deadline | Auto-calculated or manual |
| S | Court Deadlines | JSON array of bespoke deadlines |
| T | Matter Folder | Subfolder name for client files |
| U | Other Parties | For conflict check coverage |

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
