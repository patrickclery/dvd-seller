# Roadmap: DVD Seller

## Overview

Two halves joined by one file. Phase 1 locks the shared zod contract and seeds `data/catalog.json`, then puts a filterable poster grid on top of it so the visitor product is renderable from day one. Phase 2 finishes the visitor product (Plex-style detail pages, sale block, one-tap mailto) and ships it live at `patrickclery.com/dvd-seller/` through the sibling site's proven Actions workflow. Phase 3 builds the deterministic, CLI-testable ingestion scripts against the same schema, with a golden set proving remakes are surfaced rather than silently matched. Phase 4 wraps those scripts in the Claude Code skill that reads spine photos and fills the real catalog. Phase 5 is a pure deployment change: mount the proven build at `patrickclery.com/sale/dvds` from the main site's workflow.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Data Contract & Browsable Grid** - Shared zod schema, seeded catalog, validated build, and a phone-first filterable poster grid under a configurable base path
- [ ] **Phase 2: Detail Pages, Sale Flow & Live Deploy** - Deep-linkable Plex-style movie pages with sale block and mailto CTA, live at `patrickclery.com/dvd-seller/` via the mirrored Actions workflow
- [ ] **Phase 3: Ingestion Scripts** - Search, enrich, dedupe, mark-sold, and atomic validated writes from the CLI, with a remake golden test set
- [ ] **Phase 4: Ingestion Skill & Catalog Fill** - Claude Code skill that reads spine photos, batches confirmations, and loads the real collection
- [ ] **Phase 5: Mount at /sale/dvds** - Main-site compose job triggered by `repository_dispatch`, serving the catalog at its intended URL

## Phase Details

### Phase 1: Data Contract & Browsable Grid

**Goal**: A buyer can open the site (locally, under any base path) and filter, sort, and search a poster grid of seed DVDs, backed by one schema-validated catalog file that both the SPA and the ingestion scripts will share
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: DATA-01, DATA-02, DATA-03, DATA-04, DATA-05, DATA-06, DATA-07, DATA-08, CAT-01, CAT-02, CAT-03, CAT-04, CAT-05, CAT-06, CAT-07, CAT-08, CAT-09, CAT-10, CAT-11, CAT-12, CAT-13, SALE-04, DEP-02
**Success Criteria** (what must be TRUE):
  1. Visitor on a 360px-wide phone sees a 2-column poster grid (5-6 columns on desktop) of every available seed DVD showing poster, title, year, rating badge, and price/condition chips; posters load lazily from `image.tmdb.org` in fixed 2:3 boxes with no layout shift, and a title-text placeholder appears where TMDB has no poster
  2. Visitor can combine multi-select genre, decade, rating-threshold, and availability filters with sort (popularity default, IMDb rating, year, title) and instant diacritics-insensitive title search; the URL query string mirrors the state so reloading, sharing, or back-navigating reproduces the same view, and "Showing X of N", clear-all, and the empty-state message respond accordingly
  3. Sold DVDs are hidden by default and, when Availability is set to Sold or All, render desaturated with a SOLD ribbon; the header shows total and available counts plus the configured seller blurb
  4. `npm run build` fails loudly when `data/catalog.json` contains a malformed entry or a duplicate slug/tmdbId, and `NEXT_PUBLIC_BASE_PATH=/sale/dvds npm run build` served from a nested folder renders the grid with every asset and URL resolving through `withBasePath()`
  5. The browser network tab shows only `image.tmdb.org` image loads at runtime (zero metadata API calls); the repo contains no `.env*`, `.cache/`, photo directories, or API keys, none appear in the built bundle, and a no-op rewrite of the catalog produces an empty git diff

**Plans:** 4 plans

Plans:
**Wave 1**
- [ ] 01-01-PLAN.md — Walking skeleton: scaffold + zod contract + TMDB seed + one-path grid with URL search under `/sale/dvds` (legitimacy + D-01 gates)

**Wave 2** *(blocked on Wave 1 completion)*
- [ ] 01-02-PLAN.md — Contract hardening: schema/text/writer/validate-CLI specs, ESLint 9 + Prettier, `.env.example`
- [ ] 01-03-PLAN.md — Browse controls: filters/sort/search pure functions + toolbar, chips, mobile sheet, count, Clear all, empty state
- [ ] 01-04-PLAN.md — Card visuals & sale state: rating badge, price/condition chips, placeholder poster, SOLD/Reserved treatments

**UI hint**: yes
**Research**: no (Next.js static export, `generateStaticParams`, `basePath`, Suspense + `useSearchParams` are official-docs territory mirrored by the sibling repo; TMDB non-commercial posture already recorded in PROJECT.md Key Decisions)

### Phase 2: Detail Pages, Sale Flow & Live Deploy

**Goal**: Every DVD has a deep-linkable Plex-style page on the live `patrickclery.com/dvd-seller/` site where a buyer can see the full details, preview the link in a messaging app, and contact the seller in one tap
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: DTL-01, DTL-02, DTL-03, DTL-04, DTL-05, DTL-06, DTL-07, DTL-08, SALE-01, SALE-02, SALE-03, DEP-01, DEP-03, DEP-04, DEP-06
**Success Criteria** (what must be TRUE):
  1. Visitor can cold-load `patrickclery.com/dvd-seller/movie/{slug}/` and see a Plex-style hero (backdrop with gradient, poster, title, year, runtime, genres, certification, tagline), synopsis, director, 8-12 top-billed cast with headshots or initials fallback and character names, and an IMDb rating labelled by source (OMDb, or TMDB when OMDb is absent) that links to the movie's IMDb page
  2. Visitor sees the sale block (price or "Ask", condition, format, region, edition, availability) and one tap opens a prefilled `mailto:` with subject "Interested in: {Title} ({Year})" and the deep link in the body, using seller contact from config; sold titles keep a live page with a SOLD banner and a disabled CTA
  3. Pasting a detail URL into a messaging app renders a preview with title, poster, and price/status; an unknown slug lands on the static 404 page; the TMDB attribution notice and logo appear in the root layout on every page
  4. Back-navigating from a detail page returns the visitor to the grid with their previous filters, sort, and search intact
  5. Pushing to `main` runs an Actions workflow (Node 22, `npm ci`, validate, matrix build for `""` and `/sale/dvds`, link check on `out/`, `.nojekyll` assertion, leaked-key grep) and publishes to the repo's own project Pages site; the repo contains no `CNAME` and the site is not linked from the patrickclery.com homepage

**Plans**: TBD
**UI hint**: yes
**Research**: no (verbatim copy of a workflow already running in production; detail-page patterns are standard App Router static export)

### Phase 3: Ingestion Scripts

**Goal**: Seller can turn a typed list of title/year guesses into fully enriched, deduplicated, schema-valid catalog entries from the CLI, with remakes surfaced as ambiguous rather than silently matched
**Mode:** mvp
**Depends on**: Phase 1 (shared schema); can run in parallel with Phase 2
**Requirements**: ING-01, ING-02, ING-03, ING-04, ING-05, ING-06, ING-07, ING-08
**Success Criteria** (what must be TRUE):
  1. Seller runs the search script with a title/year list and receives ranked TMDB candidates with match scores; a known year scopes the search via `primary_release_year`, and same-title collisions (e.g., The Thing 1982 vs 2011) are flagged ambiguous instead of the first result being auto-selected
  2. Seller runs the enrich script for a `tmdbId` and the catalog gains a complete entry (credits, external IDs, videos, release dates, similar, optional OMDb `imdbRating`) from one TMDB call; re-enriching an existing entry preserves slug, sale, and disc fields
  3. Re-ingesting a title already in the catalog (same `tmdbId` + edition) offers to increment quantity or skip instead of creating a duplicate; seller can mark an entry sold (status + `soldAt`) or edit seller-owned fields via script without hand-editing JSON
  4. Any write that fails schema validation is refused and the catalog is never left partially written (atomic write); output ordering and key order are deterministic
  5. The golden test set of 15+ hard titles passes with every remake pair reported ambiguous; all TMDB/OMDb calls are throttled (concurrency <= 4, 429 backoff) and cached under gitignored `.cache/`; the OMDb daily-budget guard trips before the free-tier limit; API keys are read only from environment variables and never reach the browser bundle

**Plans**: TBD
**Research**: no (TMDB/OMDb endpoints, `append_to_response`, throttling and disk caching are fully specified in research/STACK.md and research/ARCHITECTURE.md)

### Phase 4: Ingestion Skill & Catalog Fill

**Goal**: Seller points the skill at a folder of shelf photos and ends up with the real collection in the catalog, having reviewed only the uncertain reads
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: SKILL-01, SKILL-02, SKILL-03, SKILL-04, SKILL-05, SKILL-06, SKILL-07
**Success Criteria** (what must be TRUE):
  1. Seller invokes the skill with one or many spine photos (file paths or a directory outside the repo) and gets per-spine structured reads (`rawText`, `titleGuess`, `yearGuess`, `editionCues`, `confidence`), including for text rotated 90 degrees either way and multiple spines per photo
  2. Only high-confidence, unambiguous matches are auto-accepted; everything else is presented in one batched confirmation (pick candidate / skip); nothing is guessed silently
  3. Seller can run a dry-run showing spine text -> matched title (year) -> confidence with zero writes, and can pass batch defaults (condition, price, format, region) that apply to every item with per-item override
  4. Each run ends with an acceptance-rate summary (auto-accepted / confirmed / skipped) and `rawText` is kept on each entry for audit
  5. Another seller can follow the README to install the skill in a clean directory and run a dry-run; the catalog path resolves from argument, `DVD_CATALOG_PATH`, or the project default, with no Patrick-specific constants or keys

**Plans**: TBD
**Research**: yes (vision prompt design for rotated, glare-prone, many-spines-per-photo input is the least documented part of the project; spike with 3-5 real shelf photos before committing the SKILL.md rubric; verify current Claude Code skill frontmatter (`allowed-tools`, `disable-model-invocation`, `${CLAUDE_SKILL_DIR}`) against live docs)

### Phase 5: Mount at /sale/dvds

**Goal**: Buyers reach the catalog at its intended address `patrickclery.com/sale/dvds` with every deep link working, rebuilt automatically whenever this repo's catalog changes
**Mode:** mvp
**Depends on**: Phase 2 (live build proven), Phase 4 (catalog populated)
**Requirements**: DEP-05
**Success Criteria** (what must be TRUE):
  1. Visitor can open `patrickclery.com/sale/dvds/` and any `/sale/dvds/movie/{slug}/` deep link; grid, filters, posters, OG previews, and mailto deep links all resolve under the new base path
  2. Pushing a catalog change to this repo's `main` fires `repository_dispatch`, and the main site's workflow checks out this repo, builds with `NEXT_PUBLIC_BASE_PATH=/sale/dvds`, copies `out/` into `out/sale/dvds/`, and redeploys, with no `CNAME` added to this repo
  3. A decision is recorded in PROJECT.md Key Decisions on whether `/dvd-seller/` stays as a preview or is disabled, and the live state matches it

**Plans**: TBD
**Research**: yes (cross-repo compose has known permission and staleness traps: fine-grained PAT scopes for `repository_dispatch`, build-from-source vs publishing an `out/` branch, 90-day artifact expiry, disabling vs keeping the `/dvd-seller/` site)

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5. Phase 3 depends only on Phase 1 and may run in parallel with Phase 2.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Data Contract & Browsable Grid | 0/4 | Planned | - |
| 2. Detail Pages, Sale Flow & Live Deploy | 0/TBD | Not started | - |
| 3. Ingestion Scripts | 0/TBD | Not started | - |
| 4. Ingestion Skill & Catalog Fill | 0/TBD | Not started | - |
| 5. Mount at /sale/dvds | 0/TBD | Not started | - |

---
*Roadmap created: 2026-09-21*
*Granularity: coarse (5 phases). Coverage: 54/54 v1 requirements mapped.*
