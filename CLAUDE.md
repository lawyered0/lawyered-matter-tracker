# Matter Tracker — Claude Code Context

This project is a legal matter tracking system backed by an Excel spreadsheet (`matter-tracker.xlsx`). It tracks open and closed client matters, limitation periods, court deadlines, conflict checks, and chronological timelines — all from a single `.xlsx` file in this directory.

## Where Things Live

- **Tracker spreadsheet**: `matter-tracker.xlsx` in this directory (the CWD)
- **Client folders**: sibling directories of this file, one per matter (e.g., `./Smith v Jones/`)
- **Per-matter briefs**: `_matter-brief.md` inside each client folder (created/updated by the `work-on-matter` skill)

## Available Skills

| Skill | Purpose |
|-------|---------|
| `daily-triage` | Scan Gmail for new emails, match to open matters, surface urgent items, present a prioritised triage summary |
| `matter-tracker` | Open, update, and close matters — pulls from Gmail + client folders to build timelines, runs conflict checks, maintains the spreadsheet |
| `work-on-matter` | Load context for an existing matter at session start so you can pick up where you left off |

## Configuration

Fill in the values below for your firm and jurisdiction. These are read by the skills at runtime.

### Firm

```
FIRM_NAME: [Your Firm Name]
LAWYER_SHORTHAND: [Your Initials]
```

`LAWYER_SHORTHAND` is used in timeline entries (e.g., "AB spoke with client re: settlement").

### Court / Tribunal Email Domains

Emails from these domains are flagged as **urgent** by the `daily-triage` skill. Add the domains used by courts, tribunals, and regulatory bodies in your jurisdiction.

```
COURT_EMAIL_DOMAINS:
  - court.gov.example
  - tribunal.gov.example
  - registry.example.gov
```

### Limitation Statutes

The statutes and default periods used for limitation deadline tracking. Edit to match your jurisdiction.

```
LIMITATION_STATUTES:
  - name: General limitation
    period_years: 2
    description: Default limitation period for most civil claims
  - name: Property damage
    period_years: 2
    description: Damage to property
  - name: Contract (written)
    period_years: 6
    description: Breach of written contract
  - name: Personal injury
    period_years: 2
    description: Bodily injury claims
```

### ID Verification

If your firm uses a third-party identity verification service, specify it here. The `matter-tracker` skill will reference this when prompting for client ID checks.

```
ID_VERIFICATION_SERVICE: [e.g., Verified.Me, Jumio, or "manual" for in-person verification]
```
