# AgoraDMV

Tools for tracking, comparing, and analyzing U.S. congressional appropriations bills.

## Projects

| Project | Type | What it does |
|---|---|---|
| [BillTrax](https://github.com/AgoraDMV/BillTrax) | Next.js web app | Dashboard for tracking bills across legislative stages, comparing versions side-by-side, and tagging sections with AI. SQLite-backed with Auth.js authentication. |
| [civic_tech_appropriations_bills](https://github.com/willhea/civic_tech_appropriations_bills) | Python library & CLI | Downloads bill XML from Congress.gov, normalizes it into a section tree, diffs two versions, and outputs JSON or standalone HTML comparison reports. |

## Architecture

```
BillTrax (Next.js)  ──►  Congress.gov API v3
civic_tech_appropriations_bills (Python CLI / library)  ──►  Congress.gov API v3
```

The Python library is a standalone, independently testable tool. BillTrax uses a local SQLite database for user accounts and saved bill state.
