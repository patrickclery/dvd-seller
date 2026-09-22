# Requirements: DVD Seller

**Defined:** 2026-09-21
**Core Value:** Turn a stack of DVD spine photos into a browsable, filterable, shareable catalog with zero hosting cost — a buyer can find a movie they want in seconds, and the seller never hand-enters a title.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Data Contract

- [ ] **DATA-01**: Catalog is a single committed `data/catalog.json` validated by one shared zod schema (`src/lib/schema.ts`) that both the SPA and the ingestion scripts import
- [ ] **DATA-02**: Each entry is keyed by `tmdbId` and carries a slug (`{tmdbId}-{kebab-title}`) generated once at ingest and never regenerated
- [ ] **DATA-03**: Each entry stores TMDB metadata (title, year, runtime, genres, certification, tagline, overview, poster/backdrop path fragments, popularity, TMDB rating, `imdbId`, director, top-billed cast with headshot paths and character names, trailer key, similar IDs) plus optional OMDb `imdbRating`
- [ ] **DATA-04**: Each entry stores seller-owned fields separate from metadata: sale status (available/sold), optional price, condition, format (DVD/Blu-ray/4K), region, edition, quantity, `addedAt`, `soldAt` — re-enrichment never overwrites these
- [ ] **DATA-05**: Each entry records provenance (`fetchedAt`, `source`, `confirmedBy`) so the TMDB 6-month cache rule can be honored and seller confirmations are never re-asked
- [ ] **DATA-06**: A `validate` script (schema + slug/tmdbId uniqueness) runs as `prebuild` so a malformed entry fails the build loudly
- [ ] **DATA-07**: Catalog file is written with deterministic ordering and key order so git diffs show only real changes
- [ ] **DATA-08**: Repo hygiene prevents leaks: `.gitignore` covers `.env*`, `.cache/`, photo directories and image globs outside `public/`; no API key ever appears in client bundles

### Catalog Browsing

- [ ] **CAT-01**: Visitor sees a responsive, phone-first poster grid (2 columns at 360px, up to 5–6 at desktop) showing poster, title, year, and rating badge for every available DVD
- [ ] **CAT-02**: Poster images are hotlinked lazily from TMDB's image CDN with a fixed 2:3 aspect box (no layout shift) and a title-text placeholder when no poster exists
- [ ] **CAT-03**: Visitor can filter by genre (multi-select)
- [ ] **CAT-04**: Visitor can sort by popularity (default), IMDb rating, year, and title
- [ ] **CAT-05**: Visitor can filter by rating threshold (e.g., 7+, 8+)
- [ ] **CAT-06**: Visitor can filter by decade
- [ ] **CAT-07**: Visitor can filter by availability (Available / Sold / All), defaulting to Available
- [ ] **CAT-08**: Visitor can search by title instantly (client-side, diacritics-insensitive, no submit button)
- [ ] **CAT-09**: Filters, sort, and search compose and are persisted in the URL query string so filtered views are shareable and survive back navigation
- [ ] **CAT-10**: Grid shows a result count ("Showing X of N"), a clear-all-filters control, and an empty-state message when nothing matches
- [ ] **CAT-11**: Sold items render with a visible SOLD treatment (desaturated + ribbon) when shown
- [ ] **CAT-12**: Header shows total and available counts plus a configurable seller blurb (private sale, pickup/shipping terms, currency)
- [ ] **CAT-13**: Site makes zero metadata API calls at runtime; the only third-party requests are visitor-browser image loads from the TMDB CDN

### Movie Detail Page

- [ ] **DTL-01**: Every movie has a statically exported, deep-linkable page at `/movie/{slug}/` generated from the catalog at build time (works on GitHub Pages with trailing slashes; unknown slugs hit a static 404)
- [ ] **DTL-02**: Detail page shows a Plex-style hero: backdrop with gradient overlay, poster, title, year, runtime, genres, certification, tagline
- [ ] **DTL-03**: Detail page shows synopsis and director
- [ ] **DTL-04**: Detail page shows top-billed cast (8–12) with headshots, names, and character names, with an initials fallback when no headshot exists
- [ ] **DTL-05**: Detail page shows the IMDb rating (OMDb-sourced when available, otherwise TMDB rating clearly labelled as such) and links out to the movie's IMDb page via `imdbId`
- [ ] **DTL-06**: Detail page has Open Graph / Twitter meta (title, poster, price/status) so shared links preview correctly in messaging apps
- [ ] **DTL-07**: Visitor can return to the grid with their previous filter state intact
- [ ] **DTL-08**: Root layout includes TMDB attribution notice and logo per TMDB terms

### Sale & Contact

- [ ] **SALE-01**: Detail page shows a sale block with price (or "Ask" when null), condition, format, region, edition, and availability
- [ ] **SALE-02**: Visitor can contact the seller with one tap via a prefilled `mailto:` (subject "Interested in: {Title} ({Year})", body includes the deep link); seller contact comes from config, not hard-coded
- [ ] **SALE-03**: Sold items keep their page live with a sold banner and a disabled CTA (deep links outlive the sale)
- [ ] **SALE-04**: Price and condition chips are visible on grid cards

### Ingestion Scripts

- [ ] **ING-01**: Seller can run a search script that takes a list of title/year guesses and returns ranked TMDB candidates with match scores, using `primary_release_year` when a year is known and flagging same-title collisions (remakes) as ambiguous — never auto-selecting the first result when ambiguous
- [ ] **ING-02**: Seller can run an enrich script that fetches full metadata for a `tmdbId` in one TMDB call (`append_to_response=credits,external_ids,videos,release_dates,similar`), optionally adds OMDb `imdbRating`, and merges into the catalog preserving seller-owned fields
- [ ] **ING-03**: All TMDB/OMDb calls are throttled (concurrency ≤ 4, 429 backoff), cached on disk (gitignored), and OMDb has a daily-budget guard so free-tier limits are never exceeded
- [ ] **ING-04**: Scripts dedupe on `(tmdbId, edition)`; on a hit they offer to increment quantity or skip rather than creating a duplicate
- [ ] **ING-05**: Seller can mark an entry sold (sets status and `soldAt`) or edit seller-owned fields via a script without hand-editing JSON
- [ ] **ING-06**: Scripts write the catalog atomically and refuse to write any entry that fails schema validation
- [ ] **ING-07**: A golden test set of ≥15 hard titles (remakes, sequels, roman numerals, box sets) verifies that all remake pairs surface as ambiguous rather than silently matching
- [ ] **ING-08**: API keys come only from environment variables; no key is ever committed or shipped to the browser

### Ingestion Skill

- [ ] **SKILL-01**: Seller can invoke a Claude Code skill with one or many spine photos (file paths or a directory outside the repo)
- [ ] **SKILL-02**: The skill reads multiple spines per photo, including text rotated 90° either way, producing per-spine structured output (`rawText`, `titleGuess`, `yearGuess`, `editionCues`, `confidence`)
- [ ] **SKILL-03**: The skill auto-accepts only high-confidence, unambiguous matches and presents everything else in one batched confirmation (pick candidate / skip) rather than guessing silently
- [ ] **SKILL-04**: The skill supports a dry-run/preview mode showing spine text → matched title (year) → confidence before any write
- [ ] **SKILL-05**: The skill accepts batch defaults (condition, price, format, region) applied to every item in a run, with per-item override
- [ ] **SKILL-06**: The skill reports an acceptance-rate summary per run (auto-accepted / confirmed / skipped) and keeps `rawText` for audit
- [ ] **SKILL-07**: The skill is portable: catalog path resolves from argument, `DVD_CATALOG_PATH`, or project default; no seller-specific constants; README explains installation for another seller

### Deployment

- [ ] **DEP-01**: Project lives in its own GitHub repo `patrickclery/dvd-seller` with a GitHub Actions workflow mirroring the sibling site (Node 22, `npm ci`, validate, build, upload `out/`, deploy-pages)
- [ ] **DEP-02**: Base path is configurable via `NEXT_PUBLIC_BASE_PATH` and every non-`Link` URL (images, OG meta, fetches) goes through one `withBasePath()` helper
- [ ] **DEP-03**: CI builds in a matrix of base paths (`""` and `/sale/dvds`), runs a link checker on `out/`, asserts `.nojekyll` is present, and greps `out/` for leaked API keys
- [ ] **DEP-04**: v1 is live as the repo's own project Pages site (`patrickclery.com/dvd-seller/`), unlinked from the homepage
- [ ] **DEP-05**: The build is mounted at `patrickclery.com/sale/dvds` by the main site's workflow (checkout this repo, build with `NEXT_PUBLIC_BASE_PATH=/sale/dvds`, copy into `out/sale/dvds/`), triggered by `repository_dispatch` from this repo, with a decision recorded on whether the `/dvd-seller/` URL stays as preview or is disabled
- [ ] **DEP-06**: No `CNAME` file exists in this repo (it would collide with the main site's domain)

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Discovery

- **DISC-01**: Visitor can search by actor or director name
- **DISC-02**: Detail page shows "More in this collection" (TMDB similar IDs intersected with the catalog, fallback shared genre + decade)
- **DISC-03**: Detail page links out to trailer (YouTube link, no iframe), TMDB, and Letterboxd
- **DISC-04**: Grid shows a "Recently added" row
- **DISC-05**: Grid supports format/region/edition badges and filters
- **DISC-06**: Box sets are a distinct `set` entry type showing as one card with an "N films" badge
- **DISC-07**: Visitor can build a multi-title inquiry list (localStorage) and send it as one prefilled `mailto:`
- **DISC-08**: Density toggle and `/` keyboard shortcut for search

### Ingestion Extras

- **INGX-01**: `--refresh` re-fetches metadata for existing entries by `tmdbId`, preserving seller fields
- **INGX-02**: Rotate-and-reread vision fallback for low-confidence spines
- **INGX-03**: Skill packaged as a plugin with example photos and config template for other sellers

### Assets

- **IMG-01**: Opt-in local image cache mode (`NEXT_PUBLIC_IMAGE_MODE=local`) fetches posters/headshots at build time into `out/` (never committed, never Git LFS) so the site can be fully self-contained

### Collection

- **COLL-01**: Collection stats page (counts by genre and decade)

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Cart / checkout / payments | Requires a backend or third-party commerce; violates zero-hosting and pushes TMDB usage into "commercial" |
| User accounts, login, saved lists synced across devices | No backend to store them |
| Comments, reviews, user ratings | No backend; spam surface; IMDb link covers it |
| Live TMDB/OMDb calls from the visitor's browser | Exposes API key, rate-limits the seller's key, breaks on rotation; data is baked at ingest |
| Scraping imdb.com for posters, ratings, or data | Violates IMDb terms; gets blocked; TMDB provides `imdbId` for link-out |
| Storing poster images in the repo by default | Bloats repo and Pages artifact; TMDB CDN hotlinking is the documented pattern; local cache is opt-in v2 |
| Embedded YouTube trailer iframes | Third-party cookies, page weight, CSP; plain link instead (v2) |
| Media playback / Plex replacement | Only the browsing/detail UX is borrowed |
| Server-side search (Algolia etc.) | A few hundred items filter in memory trivially |
| Pagination / infinite scroll | Hundreds of lazy-loaded cards render fine; pagination hides items |
| Real-time "reserved" inventory status | Needs writes from the web; seller marks sold via script and redeploys |
| Automatic pricing from eBay comps | eBay API gated; scraping brittle |
| Per-disc photos of the actual item | Contradicts no-hosted-images; condition chip + notes instead |
| Fully automatic ingestion with no review step | Vision misreads (remakes, sequels) would ship wrong products |
| Analytics / tracking scripts | Cookie banners, weight, privacy; unlisted page has tiny traffic |
| Ads or affiliate links | Would make TMDB usage commercial under their terms |
| Linking from the patrickclery.com homepage | User wants it unlisted for now |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| DATA-01 | Phase 1 | Pending |
| DATA-02 | Phase 1 | Pending |
| DATA-03 | Phase 1 | Pending |
| DATA-04 | Phase 1 | Pending |
| DATA-05 | Phase 1 | Pending |
| DATA-06 | Phase 1 | Pending |
| DATA-07 | Phase 1 | Pending |
| DATA-08 | Phase 1 | Pending |
| CAT-01 | Phase 1 | Pending |
| CAT-02 | Phase 1 | Pending |
| CAT-03 | Phase 1 | Pending |
| CAT-04 | Phase 1 | Pending |
| CAT-05 | Phase 1 | Pending |
| CAT-06 | Phase 1 | Pending |
| CAT-07 | Phase 1 | Pending |
| CAT-08 | Phase 1 | Pending |
| CAT-09 | Phase 1 | Pending |
| CAT-10 | Phase 1 | Pending |
| CAT-11 | Phase 1 | Pending |
| CAT-12 | Phase 1 | Pending |
| CAT-13 | Phase 1 | Pending |
| DTL-01 | Phase 2 | Pending |
| DTL-02 | Phase 2 | Pending |
| DTL-03 | Phase 2 | Pending |
| DTL-04 | Phase 2 | Pending |
| DTL-05 | Phase 2 | Pending |
| DTL-06 | Phase 2 | Pending |
| DTL-07 | Phase 2 | Pending |
| DTL-08 | Phase 2 | Pending |
| SALE-01 | Phase 2 | Pending |
| SALE-02 | Phase 2 | Pending |
| SALE-03 | Phase 2 | Pending |
| SALE-04 | Phase 1 | Pending |
| ING-01 | Phase 3 | Pending |
| ING-02 | Phase 3 | Pending |
| ING-03 | Phase 3 | Pending |
| ING-04 | Phase 3 | Pending |
| ING-05 | Phase 3 | Pending |
| ING-06 | Phase 3 | Pending |
| ING-07 | Phase 3 | Pending |
| ING-08 | Phase 3 | Pending |
| SKILL-01 | Phase 4 | Pending |
| SKILL-02 | Phase 4 | Pending |
| SKILL-03 | Phase 4 | Pending |
| SKILL-04 | Phase 4 | Pending |
| SKILL-05 | Phase 4 | Pending |
| SKILL-06 | Phase 4 | Pending |
| SKILL-07 | Phase 4 | Pending |
| DEP-01 | Phase 2 | Pending |
| DEP-02 | Phase 1 | Pending |
| DEP-03 | Phase 2 | Pending |
| DEP-04 | Phase 2 | Pending |
| DEP-05 | Phase 5 | Pending |
| DEP-06 | Phase 2 | Pending |

**Coverage:**
- v1 requirements: 54 total
- Mapped to phases: 54
- Unmapped: 0 ✓

---
*Requirements defined: 2026-09-21*
*Last updated: 2026-09-21 after roadmap creation (traceability mapped, count corrected 51 -> 54)*
