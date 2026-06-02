# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Monorepo layout

Two independent projects share this repo:

- `civic_tech_appropriations_bills/` — Python library + CLI for downloading and diffing U.S. bill XML from Congress.gov; also the technical foundation for a Capitol Staffer browser tool (see below)
- `BillTrax/` — Next.js 15 web app for tracking and comparing bills; includes a Node.js backend at `BillTrax/node_backend/`

They share a Congress.gov API key but are otherwise independent. **They are not coupled in code, data model, or deployment.** Concepts flow from `civic_tech_appropriations_bills` toward BillTrax as design inspiration — not as shared libraries or copy-paste.

For the full rationale, see [`docs/project-relationship.md`](docs/project-relationship.md).  
For the BillTrax build roadmap, see [`docs/billtrax-roadmap.md`](docs/billtrax-roadmap.md).

---

## Project Relationship at a Glance

| | civic_tech_appropriations_bills | BillTrax |
|---|---|---|
| Audience | Capitol Hill staffers | External orgs, researchers, accountability-focused citizens |
| Data | Private draft PDFs + public XML | Public Congress.gov data + uploaded public PDFs |
| Runtime constraint | **Browser-only after first load** (no server calls) | Fully server-backed — no offline constraint |
| Bill data model | In-memory, stateless | **Shared catalog** — bills are public infrastructure; users subscribe |
| Maturity | Technically advanced (tested diff engine) | UI-complete scaffold, data layer being built out |
| Diff approach | Python: structural XML → BillTree → diff | TypeScript: server-side section parse → DB-cached diff |

Congressional Tech's algorithms are the reference point for how BillTrax should eventually handle structural diffing and financial extraction — but the implementation must be re-done server-side in TypeScript to take advantage of persistent storage, background jobs, and LLM calls that are unavailable in a browser-only context.

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

### Commands (Docker-first — host needs only docker + git)

The web container bind-mounts the canonical Windows host path `C:/Users/enact/Projects/AgoraDMV/BillTrax` into `/app`, with polling enabled so Next.js HMR picks up host edits across Docker Desktop's Windows→Linux bind mount. No `docker compose watch` needed; do NOT use `docker cp` as a workaround. (If porting to another machine, update the volume path in `docker-compose.yml` to the local Windows path.)

```bash
cd BillTrax
docker compose up -d                          # start web (3002) + node-backend (3001)
docker compose exec web npm run typecheck     # type-check
docker compose exec web npm test              # Vitest unit tests
docker compose -f docker-compose.yml -f docker-compose.test.yml \
  run --rm playwright npm run test:e2e        # Playwright E2E (ephemeral DB)
docker compose run --rm web npm run ingest -- --congress 119 --bill hr-4366
docker compose run --rm web npm run build     # production build check
docker compose exec web npm run lint          # ESLint (run inside Docker, not on host)
```

### Environment

`BillTrax/.env.local` (copy from `.env.example`):
```
AUTH_SECRET=        # Generate with: npx auth secret
DATABASE_URL=mysql://billtrax:billtrax@mysql:3306/billtrax
MYSQL_ROOT_PASSWORD=rootpassword
MYSQL_DATABASE=billtrax
MYSQL_USER=billtrax
MYSQL_PASSWORD=billtrax
CONGRESS_API_KEY=   # Same key as Python project; optional (falls back to DEMO_KEY)
NODE_BACKEND_URL=http://node-backend:3000
AI_PROVIDER=anthropic   # anthropic | openai | deepseek
AI_MODEL=claude-sonnet-4-6
AI_API_KEY=
AI_BASE_URL=            # Only needed for DeepSeek or custom endpoint
PRESS_RELEASE_FEEDS=    # Comma-separated RSS URLs; defaults to House + Senate Appropriations
```

### Architecture

**Auth:** Auth.js v5 credentials provider, JWT sessions, bcryptjs password hashing. `middleware.ts` redirects unauthenticated requests to `/login` for all `/dashboard` routes. `auth.ts` at repo root holds the config.

**Database:** MySQL 8.4 via `mysql2/promise` (async pool). Connection pool in `src/lib/db.ts`. Schema is auto-created on first run via versioned migrations — tables: `users`, `bills`, `bill_versions`, `user_subscriptions`, `notes`, `bill_topics`, `bill_sections`, `section_diffs`, `section_diff_items`, `financial_changes`. Queries use parameterized statements. `src/lib/bills.ts` and `src/lib/users.ts` hold all query functions. The `mysql-init.sql` file at repo root is mounted into the MySQL container for first-run init.

**Node backend (`node_backend/`):** Lightweight HTTP server (no framework) that fetches bill versions from Congress.gov and returns section-level diffs. It loads `.env.local` from the repo root, so the API key is shared. The Next.js frontend calls it at runtime for live bill comparison.

**UI:** Tailwind CSS + shadcn/ui (Radix UI primitives). Use `cn()` from `src/lib/utils.ts` for class merging. CSS variables in `globals.css` define the full color and typography system — do not hardcode colors. Fonts: Fraunces (serif, headings), Inter (sans, body), JetBrains Mono (bill numbers, code).

**Key types:** `src/types/index.ts` defines `Bill`, `BillVersion`, `Amendment`, `TopicTag`. `src/lib/stages.ts` defines `BillStage`. `src/lib/congress-api.ts` defines Congress.gov API response types.

---

### Maintenance conventions (apply after every major phase of development)

**1. Security audit — always run after each phase:**
```bash
docker compose exec web npm audit
```
Fix any high/critical findings immediately. For moderate findings: assess whether the vulnerable code path is reachable in BillTrax, document the reason if deferring. The 3 remaining "moderate" items as of Phase 7 are transitive postcss-inside-next false positives (npm suggests downgrading to next@9 — do not do this; they resolve when Next.js 16 is adopted).

**2. README — keep it current:**
After each phase, update `BillTrax/README.md` to reflect:
- New features added to "Explore the features" section
- Any new env vars needed
- Any new CLI commands or scripts

**3. `.gitignore` audit — review before any commit:**
After each phase, scan for files that should not be public on GitHub. Present a text suggestion for the user to review — do not auto-edit `.gitignore`. Common candidates:
- New `*.env` or secret config files
- New build artifacts or large generated outputs
- Tool-specific caches (`.turbo/`, `.eslintcache`, etc.)
- Test output directories
- Any files containing API keys, tokens, or credentials

Current `.gitignore` items that may need addition in future phases: `.DS_Store`, `npm-debug.log*`, `coverage/`, `*.log`.
