# civic_tech_appropriations_bills — Technical Overview

How the system works at a computer science level.

---

## The core problem

A congressional bill exists across many versions — introduced, amended in committee, passed by one chamber, amended by the other, enrolled, and signed. Each version is a separate document. The challenge is: given two versions of the same bill, what exactly changed?

This is harder than it sounds because bill text isn't a flat document. It's deeply nested — divided into divisions, titles, subtitles, parts, and sections — and sections can be added, removed, modified, or physically moved to a different part of the document between versions. A naive line-by-line diff misses all of that structure.

The system solves this in four stages: **fetch → parse → diff → render**.

---

## Stage 1: Fetch (`fetch_bills.py`)

### What it does
Downloads bill XML files from the Congress.gov REST API and saves them to disk.

### Key concepts

**HTTP with retry logic.** The API has rate limits (30 requests/hour on the free key). The `api_get()` function wraps every request with 3 retry attempts, exponential backoff on `429 Too Many Requests` and `5xx` server errors, and a hard stop on `4xx` client errors.

**Pagination.** Congress.gov returns lists in pages (20 items per page). `fetch_all_committee_bills()` follows `pagination.next` URLs until it has all results, accumulating them into a single list.

**Structured file storage.** Bills are saved to `bills/<congress>-<type>-<number>/<index>_<version-slug>.xml`. The index prefix (01, 02, ...) preserves chronological order when the directory is listed alphabetically.

---

## Stage 2: Parse (`bill_tree.py`)

### What it does
Converts a bill XML file into a normalized, flat list of section nodes — a `BillTree`.

### The data structure: `BillNode`

Each section of a bill becomes a `BillNode` — an immutable record (Python frozen dataclass) with:

- **`body_text`** — the full text content of the section
- **`match_path`** — a normalized tuple of strings identifying where this section lives in the document hierarchy (e.g., `("department of defense", "military construction", "sec. 101")`)
- **`display_path`** — the same path but with original capitalization and division labels, for human display
- **`division_label`** — which division the section belongs to (e.g., "Division A: Military Construction"), if any

The `BillTree` is just metadata (congress number, bill type, version) plus an ordered list of `BillNode`s.

### Why flatten the tree?

Bill XML is hierarchical — divisions contain titles, titles contain subtitles, subtitles contain parts, etc. But for diffing purposes, only the leaf nodes (actual sections with text) matter. The hierarchy is captured in the path, not the nesting. This makes comparison straightforward: you end up with two flat lists of sections, and you pair them up.

### Two path types

The split between `match_path` and `display_path` is a deliberate design decision. When a bill goes from the House to the Senate, the same section might be renumbered or have its header capitalization changed — but it's still the same section. `match_path` normalizes away those surface differences (lowercase, whitespace collapsed) so the matcher can find the right pairing. `display_path` preserves the original for what the user sees.

### The parsing challenge: bill structure isn't uniform

Bills have three structural shapes:
1. **With divisions**: `legis-body > division > title > ... > section`
2. **Without divisions**: `legis-body > title > ... > section`
3. **Flat**: `legis-body > section`

And within titles, there can be subtitles, parts, chapters, and subchapters — any of which can contain sections. The parser handles this with `_walk_structural_children()`, a recursive function that descends through structural containers to reach actual section elements, threading the current path context down as it goes.

Appropriations bills add another wrinkle: they use flat-sibling tags (`appropriations-major`, `appropriations-intermediate`, `appropriations-small`) rather than nested sections. The parser handles these separately from standard `<section>` elements.

### Text normalization

Before storing text in a `BillNode`, the parser normalizes it:
- Runs of whitespace (spaces, newlines) → single space
- List markers like `(1)`, `(A)`, `(iv)` have spaces before them removed

This prevents meaningless formatting differences from showing up as content changes during the diff.

---

## Stage 3: Diff (`diff_bill.py`)

This is the most algorithmically interesting part. Given two `BillTree` objects, it produces a `BillDiff` — a structured description of every section that was added, removed, modified, unchanged, or moved.

### Step 1: Match nodes across versions (`match_nodes()`)

Before you can say something changed, you have to know which section in version A corresponds to which section in version B. This is the matching problem.

**The fast path:** If a `match_path` appears exactly once in both trees, it's an unambiguous match. Pair them directly.

**Collision groups:** If the same `match_path` appears in multiple places (e.g., "Sec. 101" exists in both Division A and Division B), they form a collision group. The resolver first tries to pair by normalized division title. Whatever's left over within a group goes through text similarity matching — find the best greedy pairing by actual content.

**Text similarity** is measured using Python's `SequenceMatcher` (part of the standard library). It computes a ratio between 0 and 1 — how similar two strings are based on the longest matching subsequences. This is the same algorithm behind most diff tools.

### Step 2: Classify each pair

Once nodes are paired, each pair gets a change type:

| Situation | Change type |
|---|---|
| Node exists only in new version | `added` |
| Node exists only in old version | `removed` |
| Node exists in both, text identical | `unchanged` |
| Node exists in both, text differs, similarity ≥ 0.4 | `modified` |
| Node exists in both, text differs, similarity < 0.4 | Split into `removed` + `added` (false match) |

The 0.4 threshold handles a common problem: the matcher sometimes pairs two sections that happen to share a path but have completely different content — for example, when a title is fully replaced between versions. Below 0.4, the match is treated as coincidental and the sections are treated as independent additions/deletions.

### Step 3: Detect moves (`reconcile_moves()`)

After initial classification, all `removed` and `added` sections are compared against each other using text similarity (threshold 0.6). A high similarity between a removed section and an added section at a different path indicates the section was moved, not independently deleted and rewritten. Matched pairs are reclassified as `moved`.

This is a greedy algorithm: it sorts all candidate pairs by similarity score and greedily claims the best matches, skipping any section once it's been matched.

### Step 4: Financial extraction

Dollar amounts embedded in bill text are extracted with a regex that finds `$X,XXX,XXX` patterns. The system also handles floor amendment annotations — text like `(increased by $2,000,000)` — by stripping them out before comparing amounts. The annotation references the original budget request, not the previous bill version, so the base number in the text is the actual appropriation.

The `match_amounts()` function aligns dollar amounts across old and new text using word-level diff positioning — it finds which words were added, removed, or kept, and uses those positions to pair up the corresponding amounts.

### Output: `BillDiff`

The diff is a list of `NodeDiff` objects. Each has:
- `change_type` — one of: `added`, `removed`, `modified`, `unchanged`, `moved`
- `old_node` and/or `new_node` — the matched `BillNode`(s)
- `financial_change` — optional, if dollar amounts changed

The diff can be serialized to JSON or passed to the HTML formatter.

---

## Stage 4: Render (`formatters/html.py`)

### What it does
Takes a `BillDiff` and produces a single, self-contained HTML file — no external CSS or JS dependencies.

### Key techniques

**Word-level diff (`word_diff()`):** For modified sections, shows exactly what words changed inline. It splits both old and new text into words, runs the sequence matcher, and wraps deletions in `<del>` tags and insertions in `<ins>` tags. This is finer-grained than line-by-line diff.

**Sidebar navigation:** Built as a scrollable list of all changed sections. Client-side JavaScript filters the list as you type (simple substring match on the visible text). Each sidebar item is a hyperlink that scrolls to the corresponding section card.

**Financial summary table:** A sortable HTML table of all sections where dollar amounts changed. JavaScript handles column sorting, respecting merged cells (`rowspan`) for grouped rows.

**Self-contained output:** All CSS and JavaScript are inlined as strings in the Python module (`_CSS` and `_JS` constants). The output file has zero external dependencies and can be opened offline or attached to an email.

---

## How the pieces connect

```
Congress.gov API
      │
      ▼
fetch_bills.py ──► bills/<congress>-<type>-<number>/<version>.xml
                                │
                                ▼
                         bill_tree.py
                         normalize_bill()
                                │
                                ▼
                    BillTree (list of BillNodes)
                   /                           \
           old version                     new version
                   \                           /
                         diff_bill.py
                         diff_bills()
                                │
                                ▼
                           BillDiff
                      (list of NodeDiffs)
                          /         \
                    to JSON       formatters/html.py
                                  format_html()
                                        │
                                        ▼
                               report.html (standalone)
```

---

## Design constraints worth knowing

**Immutability.** `BillNode` and `BillTree` are frozen dataclasses — they cannot be modified after creation. This makes them safe to cache and reuse across tests and concurrent operations without defensive copying.

**No database.** The system is stateless. Each run reads XML files from disk, does its work in memory, and writes output. There's no persistence layer to manage.

**Slow tests are explicit.** Tests that require real bill XML files (which can be large and require API downloads) are tagged `@pytest.mark.slow`. The fast suite runs entirely on inline XML snippets and takes about one second. This lets CI run quickly while still having comprehensive integration coverage.

**The `bills/` directory is a cache, not source of truth.** The XML can always be re-downloaded. Nothing in the system treats the on-disk files as authoritative — they're just the local copy of what Congress.gov has.
