# Project Relationship — Inspiration Without Coupling

> How `civic_tech_appropriations_bills` and `BillTrax` relate to each other.

---

## The Two Products

### civic_tech_appropriations_bills (Congressional Tech)

A Python library and CLI built for **Capitol Hill staffers** who need to work with appropriations bills — including private draft versions circulated before public introduction. The tool runs as a self-contained web page: once the page loads, it makes **no further server connections**. All bill content stays on the user's device. This is a hard privacy requirement because draft bills are sensitive pre-introduction documents.

At the draft stage, bills exist only as watermarked PDFs — no XML. The tool must handle PDF extraction entirely in the browser (PDF.js). After public introduction, XML versions are available from Congress.gov. The Python diff engine (bill_tree.py, diff_bill.py) is the algorithmic core: it parses XML into a flat, normalized list of sections and performs structural diffing with move detection and financial extraction.

**Constraint summary:** Browser-only after page load. PDF-first at draft stage. No server, no database. Fully offline-capable once loaded.

---

### BillTrax

A Next.js 15 web application serving as a **public records platform** — a shared catalog of congressional appropriations bills and committee reports, publicly browsable by anyone, with authenticated personal workspaces for users who want to track, annotate, and receive alerts.

Primary audience: external organizations (think tanks, government affairs professionals, federal contractors, universities, non-profits) and accountability-focused citizens. Staffer use is welcome but not the design target; staffer-specific collaboration workflows belong to the Capitol Staffer tool.

BillTrax is the **public-records complement** to the Capitol Staffer tool's private-draft model. The Capitol Staffer tool handles documents that must never leave the user's device. BillTrax handles documents that are public by definition — and makes them more accessible to people outside Congress.

BillTrax runs with a full server stack: SQLite via better-sqlite3, Auth.js sessions, Next.js API routes, a Node.js comparison backend, and the ability to call external APIs (Congress.gov, Claude) at any time. It has no offline constraint and no private-data handling requirement.

**Constraint summary:** Fully server-backed. Public data only. No offline requirement. Shared catalog, personal annotations.

---

## What Is Shared (Concepts)

These ideas come from Congressional Tech and should inform how BillTrax is built — but the implementations are independent:

| Concept | Congressional Tech approach | BillTrax approach |
|---|---|---|
| **Section extraction** | XML → `BillNode` list via `bill_tree.py` | Server-side TypeScript: parse imported text into rows in `bill_sections` table |
| **Structural diff** | `match_nodes()` — normalize paths, match sections, detect moves | Same logic re-implemented in TypeScript/Node.js, results cached in DB |
| **Financial extraction** | Regex on `body_text`, strips floor amendment annotations | Same regex approach, server-side, stored in `financial_changes` table |
| **Word-level diff** | `<del>`/`<ins>` tags in standalone HTML output | `diffWords()` in `src/lib/diff.ts`, rendered inline in the compare view |
| **Topical search** | Not yet built | Server-side Claude API call for section relevance ranking |
| **Bill+report pairing** | Combined PDF view, cross-reference by agency | Companion bill linking (House ↔ Senate), committee report as a separate version |

---

## What Is Not Shared

- **No shared code.** The Python library is not called by BillTrax. The Node.js comparison backend (`node_backend/`) is an independent reimplementation of similar ideas.
- **No shared data model.** The Python project uses frozen dataclasses; BillTrax uses SQLite rows with a completely different schema.
- **No shared deployment.** They run independently.
- **Different development tracks.** Changes to Congressional Tech's diff algorithm do not automatically need to be mirrored in BillTrax, and vice versa.

---

## Where BillTrax Goes Further

Because BillTrax is server-backed, it can do things the browser-only tool cannot:

- **Persistent section storage** — once a bill's sections are extracted, they're in the DB; diff is computed once, cached, shown instantly on subsequent visits
- **Background version polling** — a job checks Congress.gov daily for new versions of tracked bills and imports them automatically
- **LLM-powered features** — topic matching, diff summarization, funding opportunity classification via Claude API
- **Full-text search** — SQLite FTS5 across all tracked bill sections and report language
- **Multi-user workspaces** — org-level interest area tagging, shared bill lists, role-based access
- **Export and reporting** — PDF/CSV report generation, shareable read-only links for clients

---

## Where Congressional Tech Is Currently Ahead

Congressional Tech has a working, tested pipeline with capabilities BillTrax has not yet matched:

- **Structural XML parsing** — handles all three bill structural shapes (with divisions, without divisions, flat), plus appropriations-specific tag types
- **Move detection** — `reconcile_moves()` identifies when a section is relocated across the document, not just modified
- **Financial annotation handling** — correctly ignores floor amendment baseline annotations (`(increased by $2,000,000)`) when extracting dollar amounts
- **Standalone HTML output** — self-contained diff reports with sidebar navigation, sortable financial tables, and word-level inline diffs, all with zero external dependencies

BillTrax should eventually match and then exceed each of these capabilities on the server side.

---

## How to Think About Cross-Project Work

When building a feature in BillTrax that has an analogue in Congressional Tech:

1. **Read the Congressional Tech implementation first.** The Python code in `bill_tree.py` and `diff_bill.py` is the reference design. The technical overview in `docs/civic-tech-technical-overview.md` explains the reasoning.
2. **Adapt, don't port.** The Python code uses synchronous in-memory processing. BillTrax needs async TypeScript, DB persistence, and incremental computation. The logic translates; the architecture does not.
3. **Leverage what the server gives you.** If Congressional Tech does something expensive on every run because it has no cache, BillTrax should do it once and store the result.
4. **Keep them independent.** Do not create cross-project imports, shared configuration, or deployment dependencies. The shared insight is algorithmic, not structural.
