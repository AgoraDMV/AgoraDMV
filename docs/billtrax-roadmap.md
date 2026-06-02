# BillTrax — Implementation Roadmap

> Last updated: 2026-05-15  
> Working directory: `BillTrax/`

This document tracks what exists, what is broken, and what to build in priority order. Update it as phases complete.

For product scope and audiences, see `BillTrax/docs/SCOPE.md`.  
For the project relationship with `DeltaTrack`, see `docs/project-relationship.md`.  
For user stories, see `BillTrax/docs-for-ai/user-stories.md`.

---

## Architectural North Star

BillTrax is a **public-records platform** — a shared catalog of congressional appropriations bills, publicly browsable by anyone, with personal workspaces for users who want to track, annotate, and get alerts. This means:

- **Bills are shared infrastructure.** Once any bill enters the catalog (by any ingestion path), it's visible to every visitor. Processing (section extraction, diffs, financial changes) happens once and is cached for all.
- **Personal data is per-user.** Notes, topic tags, subscriptions, and interest areas belong to individual users.
- **Anonymous-first read paths.** Bill detail, section views, and diff views require no login. Auth gates only writes and the personalized dashboard.
- **Marketing and app live in the same Next.js deployment** at different route prefixes (`/` for marketing, `/dashboard` for the authenticated workspace, `/bills` for the public catalog).
- **Committee reports are a first-class paired document**, not just another bill version. Each appropriations bill has a companion committee report; the platform stitches them together by agency.

---

## Current State (as of May 2026)

### What works end-to-end

| Feature | Notes |
|---|---|
| Auth — login / register / sessions | Auth.js v5, JWT, bcrypt; login page redirects logged-in users |
| Shared catalog DB | Bills are shared infrastructure; `user_subscriptions` per user |
| Congress.gov bill import + ingest CLI | Fetches versions, XML, text, amendments, sections |
| Public catalog `/bills` | Browse + title/number filter + section full-text search (FULLTEXT) |
| Public `/bills/[id]` | Anonymous read; subscribe button gates writes |
| Shareable links `/s/[token]` | `POST /api/share` creates tokens; page resolves to bill detail |
| Dashboard bill card grid | Subscriptions-based; real DB data |
| Bill detail — Compare tab | Word-level diff; text lazy-loaded per selected version pair |
| Bill detail — Sections tab | Section-level diff; cached in `section_diffs` |
| Bill detail — Bill Text tab | Lazy-loaded via `/api/bills/[id]/versions/[versionId]` |
| Bill detail — Amendments tab | Fetched from Congress.gov on ingest; rendered with party badge |
| Bill detail — Financial tab | Per-section dollar deltas; CSV export |
| Bill detail — Agency tab | Committee report sections paired by agency label |
| Bill detail — Opportunities tab | AI classification (funding_opportunity/directive/deadline/restriction) |
| Bill detail — Year over year tab | Cross-bill word diff vs prior year base version |
| Bill detail — Lifecycle tab | Visual stage tracker with version/date annotations |
| Bill detail — Share button | Copies `/s/[token]` URL to clipboard |
| Bill detail — Export CSV button | Downloads financial summary CSV |
| Notes | Real DB persistence; shown on dashboard |
| Topic tagging | Real DB persistence; AI topic matching |
| PDF upload | Server-side extraction; `source=upload` badge |
| Committee report upload + agency view | PDF → agency block parsing → side-by-side |
| AI diff summary | Claude summarizes section diff; cached per version pair |
| Interest areas | Per-user keyword sets; management UI on dashboard |
| Dashboard activity feed | New versions + notes for subscribed bills |
| Press releases | RSS poller at `POST /api/press-releases`; bill matching by number regex |
| Full-text search | MySQL FULLTEXT on `bill_sections.body`; section excerpts with bill links |
| Marketing pages | `/`, `/how-it-works`, `/example`, `/for/*` |

### Remaining gaps

- **Hearings** — `hearing_transcripts` table exists; audio→Whisper→Claude pipeline not built (opt-in per-bill trigger, high cost) — deep backlog

### Recently completed (May 2026)

| Feature | Notes |
|---|---|
| Own-rolled SMTP | `src/lib/email.ts` — nodemailer, provider-agnostic, log-only stub when `SMTP_HOST` unset |
| Password reset flow | `/forgot-password` + `/reset-password/[token]` pages; `password_resets` table with sha256-hashed tokens |
| Account profile expansion | `users` table now has `username, phone, organization, email_verified, last_login`; `/dashboard/settings` |
| Version-change email alerts | `notifications.ts` + `notification_log` table; queued on ingest and poll |
| Congress.gov background polling | `scripts/poll-congress.ts` — `npm run poll:congress -- --congress 119` |
| Roll call votes | `roll_call_votes` + `member_votes` tables; `sync:roll-call` script; sidebar card on bill detail |
| Member bios | `members` table; `sync:members` script; party/state badge on bill sponsor display |
| Companion bill cross-compare | Verified working — `BillDetailClient.tsx` companion tab with word-level diff |
| Multi-congress recurrence | Verified working — `AgencyView` recurrence banner with count + history |
| Cross-agency topical search | Verified working — `SectionSearch.tsx` Topic mode with Claude term expansion |
| Dashboard topic alerts | Verified working — `getTopicAlerts` surfaced in dashboard sidebar card |

---

## Phase 0 — Public Records Foundation

**The architectural refactor that unlocks every later phase.**

### Data model split

The current `bills` table has a `user_id` column — bills are private per user. This needs to invert:

```
bills                    (shared catalog — no user_id)
  id, number, title, congress, chamber, bill_type, sponsor, introduced,
  status, stage, summary, related_bill_id, prev_year_bill_id, created_at

bill_versions            (shared catalog)
  id, bill_id, label, version_code, date, text, is_base, congress_url, created_at

user_subscriptions       (per-user)
  id, user_id, bill_id, baseline_version_id, created_at
```

User-specific data (notes, topics, interest areas) references `bill_id` directly.

### Background ingestion job
- Cron/worker that polls Congress.gov for all appropriations bills per Congress and ingests new bills and versions automatically.
- Uses existing `fetchBillVersions`, `fetchBillMetadata`, `downloadVersionText` from `src/lib/congress-api.ts`.
- Idempotent: skip bills/versions already in catalog.

### Anonymous read routes
- `/bills` — browse the full catalog (no login required)
- `/bills/[id]` — bill detail (no login required; "Subscribe" button prompts login)
- `/bills/[id]/compare/[fromCode]/[toCode]` — shareable diff URL (no login)

### Marketing pages (same Next.js app, root routes)
- `/` — home: value prop, who it's for, primary CTA ("Browse bills"), secondary ("Sign up")
- `/how-it-works` — pipeline: ingest → structure → diff → agency view → topical search
- `/example` — live or static showcase diff (real HR 4366 House vs. Senate data)
- `/for/local-government`, `/for/universities`, `/for/industry`, `/for/citizens` — stakeholder lenses

### Subscription flow
Replace the "import bill" flow with "subscribe to bill from catalog". Users can still PDF-upload a bill to add it to the catalog (with an "Uploaded — not official text" badge), then subscribe.

### One-time data migration
Move existing per-user bill rows into the shared catalog; create `user_subscriptions` rows to preserve their current workspace.

---

## Phase 1 — Foundation Completion

**Close the gaps in what's already scaffolded.**

### 1A. Fix auth redirect bug
Check `src/app/login/page.tsx` — redirect to `/dashboard` if user already has a valid session.

### 1B. Notes persistence
- Wire the existing `notes` DB table to real reads/writes.
- `GET /api/notes?billId=` — per-user
- `POST /api/notes` — `{ title, body, billId? }`
- Replace `mockNotes` on dashboard with real fetch.
- Note creation UI linked from dashboard sidebar.

### 1C. Topic tagging persistence
- `topics` table: `id`, `bill_version_id`, `user_id`, `name`, `created_at`
- `POST /api/topics` — save topic for a bill version
- Update `TopicDialog` to POST on save instead of updating local state

### 1D. PDF upload — server-side extraction
- Install `pdf-parse` server-side
- `POST /api/catalog/upload` — accepts `multipart/form-data`, extracts text, creates a catalog bill entry
- Badge uploaded versions as "Uploaded — not official text" everywhere they appear
- Add `source` column to `bill_versions`: `"congress"` or `"upload"`

### 1E. Gold copy UI
- "Set as baseline" button per version in the versions sidebar
- `PUT /api/bills/[id]/baseline` — sets `is_base` in DB
- `rowsToBill` in `bills.ts` should use DB flag, not always index 0

---

## Phase 2 — Section-Aware Architecture

**The core technical upgrade, all catalog-level shared data. Inspired by `bill_tree.py` in Congressional Tech — re-implemented server-side in TypeScript.**

### New tables

```sql
bill_sections (
  id, version_id, seq, path, match_path, label, body, created_at
)

section_diffs (
  id, bill_id, base_version_id, new_version_id, cached_at
)

section_diff_items (
  id, diff_id, change_type,  -- added | removed | modified | unchanged | moved
  base_section_id, new_section_id, similarity
)

financial_changes (
  id, diff_item_id, old_amount, new_amount, delta, context
)
```

### 2A. Section extraction on ingest
`src/lib/section-parser.ts` — `splitIntoSections(text, versionId)`:
- Heuristic: find `SEC. N.` / `SECTION N.` headers
- Store each section as a `bill_sections` row with normalized `match_path`
- Run automatically during bill ingestion (Phase 0 job + manual import)

### 2B. Section-level diff display
- `matchSections(baseVersionId, newVersionId)` — pairs by `match_path`, computes text similarity (LCS ratio, same approach as Python `SequenceMatcher`)
- Display as section cards: Added (green) / Removed (red) / Modified (amber, with inline word diff) / Unchanged (collapsed)
- Cache results in `section_diffs` + `section_diff_items`
- Fall back to current flat word-diff when sections unavailable

### 2C. Financial extraction
- Regex: `\$[\d,]+` on `body` field, strip floor amendment annotations (`(increased by $N)`)
- Pair amounts across base/new using word-position alignment
- Store in `financial_changes`

### 2D. Financial summary tab
- New "Financials" tab on bill detail
- Sortable table: section path / prior amount / new amount / delta
- Net change summary row

---

## Phase 2.5 — Supplemental Data Sources Investigation

**Not a build phase. A planning sprint that produces a document.**

Survey each candidate source:
- Press releases (member + committee RSS feeds)
- Committee hearing transcriptions (AI pipeline: audio → Whisper → Claude summarization)
- Member bios (unitedstates/congress-legislators GitHub dataset)
- Roll call votes (ProPublica Congress API / Congress.gov)

For each source, document:
- Data shape and what it contributes to BillTrax
- Ingestion cost (API call, scraping, AI pipeline)
- Refresh cadence
- Integration complexity estimate
- Whether it changes the data model

Pick 1–2 to commit to in Phase 6. Define the integration pattern future sources follow.

**Output:** `docs/supplemental-data-sources.md`

---

## Phase 3 — Bill + Committee Report Pairing

**First-class document pairing. The committee report explains what the bill numbers mean — users need both.**

### New table

```sql
committee_reports (
  id, bill_id, chamber,  -- 'house' | 'senate' | 'conference'
  label, date, text, source,  -- 'upload' | 'api'
  created_at
)

report_sections (
  id, report_id, seq, agency_label, body, created_at
)
```

### 3A. Report ingestion
- PDF upload path (uses Phase 1 pipeline), labeled as "House Report", "Senate Report", "Conference Report"
- Reports are linked to bills at the `bill_id` level, not version level (a report accompanies a bill, not a specific version)

### 3B. Report section extraction
- Heuristic: appropriations reports use consistent agency header patterns (e.g., "DEPARTMENT OF DEFENSE", "BUREAU OF RECLAMATION")
- Store extracted agency blocks in `report_sections`

### 3C. Agency view
- New "Agency" tab on bill detail (when a paired report exists)
- User selects an agency from a dropdown
- View shows: bill sections for that agency alongside matching report language
- Side-by-side: bill text on left, report language on right

### 3D. Cross-document compare
- "Report changes" compare mode: House report vs. Senate report for the same bill
- Uses same section diff engine, rendered with report-specific labels

### 3E. Multi-Congress report-language tracking
- Track recurring agency sections across years (match by normalized agency label)
- "This section appeared in 6 of the last 8 years" indicator

---

## Phase 4 — Intelligence Layer

**Server-side capabilities a browser-only tool can't match.**

### 4A. Full-text search
SQLite FTS5 virtual table over `bill_sections.body` + `report_sections.body`:
```sql
CREATE VIRTUAL TABLE sections_fts USING fts5(label, body, content='bill_sections');
```
`GET /api/search?q=&congress=` — returns section excerpts with match highlighting, no login required.

### 4B. Real AI topic matching
- `POST /api/topics/match` — `{ billVersionId, topicName, topicDescription }`
- Fetch sections for that version, pass top candidates to Claude for ranking
- Return top N sections with relevance scores
- Stream response; cache per `(versionId, topicName)` pair

### 4C. AI diff summary panel
- `POST /api/summarize-diff` — passes already-computed section diffs
- Returns structured summary: "Key changes", "Sections added", "Dollar amount changes", "Sections removed"
- Shown as a collapsible panel alongside the diff view; cached per version-pair

### 4D. Cross-agency topical search
One of the most-requested features. "Filter for energy across all bills" should catch electricity, power, grid, renewables.
- Semantic expansion: user enters a topic → Claude returns a set of synonym/related terms → FTS query fans out across all terms
- Results: matching sections across all catalog bills, grouped by bill + agency
- Available from the public catalog browse page, no login required

---

## Phase 5 — Product Depth

**Features that complete the core product surface.**

- Amendments from Congress.gov API — `/bill/{congress}/{type}/{number}/amendments`; populate `amendments` table; render in amendments tab with sponsor, status badge, description
- Companion bill linking UI — House ↔ Senate link action; chamber-compare mode (diff a House version against a Senate version across linked bills)
- Year-over-year compare — "link prior year" action; "What's new vs. prior year" tab using same section diff engine with labels "New this year" / "Cut" / "Amount change"
- Dashboard enrichment — activity feed (new versions, topic changes), "tracked topics with changes" list

---

## Phase 6 — Supplemental Data Source Integrations

**Discrete mini-projects. One per source identified in Phase 2.5.**

Each integration is its own planning + build sprint, following the integration pattern defined in Phase 2.5.

Likely candidates (in order of implementation cost):
- **Press releases** — RSS polling, member + committee feeds, surface on bill detail
- **Committee hearings** — audio/video transcription (Whisper) + summarization (Claude); higher cost

---

## Phase 7 — External Audience Features

**The features that serve the paying customer — think tanks, government affairs, accountability orgs.**

- Funding opportunity classification — Claude labels each bill section as `funding_opportunity` / `directive` / `deadline` / `restriction`; "Opportunities" tab on bill detail
- Interest areas tagging — per-user keyword sets; cross-bill opportunities feed when new matching sections appear
- Appropriations lifecycle tracker — kanban/table: subcommittee → full committee → House floor → Senate floor → conference → JES → enrolled; links to text at each stage
- Export — PDF/CSV reports (bill sections + financial summary + opportunities)
- Shareable public links — `/s/{token}` read-only views of specific diffs, section selections, or topic results

---

## Open Questions

1. **Section parser fidelity** — Start with regex heuristics (SEC. headers) or invest in XML-aware parsing for Congress.gov XML versions from the start? Heuristic is faster to ship; XML-aware matches Congressional Tech's accuracy for structured appropriations bills.

2. **Node backend fate** — Keep as sidecar (proxy through Next.js API routes) or inline into TypeScript modules? Option B is cleaner at scale; Option A lets the backend evolve independently while the Next.js app is still scaffolding.

3. **Catalog seeding strategy** — Seed the catalog with all 119th Congress appropriations bills on first deployment, or seed lazily as users encounter bills? Proactive seeding gives a better cold-start experience but requires more upfront work.

4. **Anonymous user behavior** — Should anonymous visitors be able to search and browse freely, or should search require login (reduces API/compute cost but hurts public-records mission)?

5. **Notes format** — Free text, or inline annotation within bill text (highlight-and-note)? Free text ships faster; inline annotation is a richer experience for the external-org audience.
