# Supplemental Data Sources — Investigation Report

> Phase 2.5 output · BillTrax  
> Last updated: 2026-05-13

Survey of candidate supplemental data sources. Evaluated on: data shape, ingestion cost, refresh cadence, integration complexity, and data model impact. Phase 6 commitments at the end.

---

## Candidate Sources

### 1. Press Releases — Member + Committee RSS Feeds ✅ Selected

**Data shape:** RSS 2.0 / Atom from member and committee official websites. Each item: title, link, pubDate, HTML body. Committee feeds focus on markups and hearings; member feeds mix constituent and policy content.

**What it adds:** Statements from sponsors/committee chairs when a bill is viewed; public-attention timeline around bill versions; "Members who mentioned this bill" cross-reference.

**Ingestion cost:** HTTP GET each feed, parse RSS — no auth, no API key. Feed URLs sourced from `unitedstates/congress-legislators` dataset (maps members to official website URLs). Storage: title + URL + pubDate + excerpt — very low.

**Refresh cadence:** Every 6–24 hours. Idempotent by `source_url` deduplication.

**Integration complexity:** Low. `fast-xml-parser` for RSS. Bill matching via regex on bill number and title in body text.

**Data model:**
```sql
press_releases (
  id, bill_id (nullable FK), source_url UNIQUE,
  feed_url, source_name, title, excerpt, published_at, created_at
)
```

---

### 2. Committee Hearing Transcriptions ✅ Selected (targeted only)

**Data shape:** Hearing audio/video → Whisper transcription → Claude summarization. Produces: title, date, committee, witness list, testimony excerpts, bill references.

**What it adds:** "Discussed in X hearings" cross-reference; witness testimony alongside bill text; committee concerns explaining why specific sections changed.

**Ingestion cost:** Whisper: ~$0.006/min; typical 3-hour hearing ≈ $1.10. Claude summarization: ~$0.03–$0.10 per hearing. Total: ~$1–$2 per hearing. Affordable per-bill, not for broad ingestion.

**Refresh cadence:** On-demand when user requests it for a tracked bill, not continuous.

**Integration complexity:** High. Requires video URL fetch, ffmpeg audio extraction, batched Whisper, Claude post-processing, retry logic. Built as a background job, not inline.

**Data model:**
```sql
hearing_transcripts (id, bill_id, title, date, committee, text, status, created_at)
hearing_mentions (id, transcript_id, bill_id, excerpt, created_at)
```

**Verdict:** Implement as opt-in per-bill trigger, not broad ingest.

---

### 3. Member Bios (unitedstates/congress-legislators) — utility enhancement

Single JSON/YAML fetch from GitHub (~1MB). Adds party badge, state, district, photo to sponsor display. Near-zero cost. Implement as a side task in Phase 5 alongside amendments (which need party badges). Not a standalone Phase 6 integration.

---

### 4. Roll Call Votes (Congress.gov) — Phase 6 third priority

Per-bill vote records: date, chamber, passage status, party breakdown, individual member votes. Requires vote-to-bill matching. High value for accountability orgs; implement when lifecycle tracker (Phase 7) is in place.

---

## Integration Pattern (all Phase 6 sources)

1. **Ingest script** — `scripts/sync-{source}.ts`, runnable via `npm run sync:{source}`
2. **Idempotent upsert** — dedup by source URL or external ID
3. **Bill matching** — regex on bill number; `bill_id` nullable until matched
4. **API route** — `GET /api/{source}?billId=` returns matched items
5. **UI surface** — sidebar card on bill detail; no new tab unless volume warrants it
6. **Refresh** — `docker compose run --rm web npm run sync:{source}`

---

## Phase 6 Commitments

| | Feature | Tables | Priority |
|---|---|---|---|
| 6A | Press releases (RSS) | `press_releases` | First |
| 6B | Hearing transcriptions (Whisper + AI, targeted) | `hearing_transcripts`, `hearing_mentions` | Second |
| 6C | Roll call votes | `roll_call_votes`, `member_votes` | Third |
