# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Monorepo layout

Two independent projects share this repo:

- `civic_tech_appropriations_bills/` — Python library + CLI for downloading and diffing U.S. bill XML from Congress.gov
- `BillTrax/` — Next.js 15 web app for tracking and comparing bills; includes a Node.js backend at `BillTrax/node_backend/`

They share a Congress.gov API key but are otherwise independent. The Python project is the foundational diff engine; BillTrax is the user-facing product.

---

## civic_tech_appropriations_bills

See `civic_tech_appropriations_bills/AGENTS.md` for full architecture details and test conventions.

### Commands

```bash
cd civic_tech_appropriations_bills
uv sync                                   # Install dependencies (run once)
uv run pytest -m "not slow"              # Fast unit tests (~1s, no bill files needed)
uv run pytest                             # All tests (requires downloaded bill XML in bills/)
uv run pytest test_diff_bill.py::TestMatchNodes::test_all_matched  # Single test
uv run ruff format .                      # Format
uv run ruff check --fix .                 # Lint

# CLI
uv run python fetch_bills.py versions 118 hr 4366          # List available versions
uv run python fetch_bills.py download 118 hr 4366           # Download all versions
uv run python diff_bill.py compare old.xml new.xml --format html -o report.html
```

### Environment

`civic_tech_appropriations_bills/.env`:
```
CONGRESS_API_KEY=your_key   # Free at api.congress.gov/sign-up/ — falls back to DEMO_KEY (30 req/hr)
```

### Architecture summary

Pipeline: **fetch → parse → diff → output**

| Module | Role |
|---|---|
| `fetch_bills.py` | Downloads bill XML from Congress.gov → `bills/<congress>-<type>-<number>/` |
| `bill_tree.py` | Parses XML → `BillTree` (list of frozen `BillNode` dataclasses) |
| `diff_bill.py` | `match_nodes()` pairs nodes across versions; `reconcile_moves()` detects section relocations |
| `formatters/html.py` | Renders `BillDiff` to standalone HTML with financial summary + word-level diffs |

**`BillNode` has two path types:**
- `match_path` — normalized (lowercase, no division label) — used for cross-version pairing
- `display_path` — original case, includes division label — used for human-readable output

**Diff thresholds:** similarity < 0.4 → false match (split into remove+add); similarity ≥ 0.6 → move candidate.

**Floor amendment note:** annotations like `(increased by $2,000,000)` reference the budget request baseline, not the previous bill version. The base amount in text IS the correct appropriation.

---

## BillTrax

### Commands

```bash
cd BillTrax
npm run dev        # Dev server at http://localhost:3000
npm run build      # Production build (also runs type-check)
npm run lint       # ESLint

# Node backend — run as a separate process
cd BillTrax/node_backend
npm start          # Bill comparison API, default port 3000
```

No tests are currently configured.

### Environment

`BillTrax/.env.local`:
```
AUTH_SECRET=        # Generate with: npx auth secret
DATABASE_URL=./billtrax.db
CONGRESS_API_KEY=   # Same key as Python project; optional (falls back to DEMO_KEY)
```

### Architecture

**Auth:** Auth.js v5 credentials provider, JWT sessions, bcryptjs password hashing. `middleware.ts` redirects unauthenticated requests to `/login` for all `/dashboard` routes. `auth.ts` at repo root holds the config.

**Database:** SQLite via `better-sqlite3` (synchronous API). Singleton connection in `src/lib/db.ts`. Schema is auto-created on first run — tables: `users`, `bills`, `bill_versions`, `notes`. Queries use `.prepare()` with parameterized statements. `src/lib/bills.ts` and `src/lib/users.ts` hold all query functions.

**Node backend (`node_backend/`):** Lightweight HTTP server (no framework) that fetches bill versions from Congress.gov and returns section-level diffs. It loads `.env.local` from the repo root, so the API key is shared. The Next.js frontend calls it at runtime for live bill comparison.

**UI:** Tailwind CSS + shadcn/ui (Radix UI primitives). Use `cn()` from `src/lib/utils.ts` for class merging. CSS variables in `globals.css` define the full color and typography system — do not hardcode colors. Fonts: Fraunces (serif, headings), Inter (sans, body), JetBrains Mono (bill numbers, code).

**Key types:** `src/types/index.ts` defines `Bill`, `BillVersion`, `Amendment`, `TopicTag`. `src/lib/stages.ts` defines `BillStage`. `src/lib/congress-api.ts` defines Congress.gov API response types.
