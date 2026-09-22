# Feature Research

**Domain:** Static, zero-hosting personal DVD-collection catalog for sale ("IMDb clone for selling DVDs") + agent-driven photo-to-catalog ingestion skill
**Researched:** 2026-09-21
**Confidence:** MEDIUM — feature landscape cross-checked against the official pages of Letterboxd, Plex, Libib, eBay item-specifics, tinyMediaManager and the TMDB API reference. Individual web sources classify LOW via the confidence seam; agreement across 5+ independent products raises the synthesis to MEDIUM. No claim below rests on a single source.

## Scope Note

Two products share one data file:

1. **Visitor SPA** — the buyer-facing catalog at `/sale/dvds`. Benchmarks: Letterboxd (grid + filters), Plex/Jellyfin (detail "preplay" page), IMDb/TMDB (metadata depth), Libib published collections (personal catalog shared by URL), eBay/Discogs listings (what a *buyer of a physical disc* needs to know).
2. **Ingestion skill** — the seller-facing Claude Code skill: photo(s) of spines → identified titles → TMDB metadata → JSON entries. Benchmarks: barcode-scan catalogers (Libib, CLZ Movies), tinyMediaManager's scrape-and-match flow, bookshelf-spine OCR pipelines (Shelf Scan, bookshelf-scanner, BookSpinesOCR).

Everything in PROJECT.md's Active Requirements is treated as in-scope and is placed in the tables below with a `[REQ]` tag.

## Feature Landscape

### Table Stakes (Users Expect These)

Missing any of these and a buyer leaves, or the seller stops using the skill.

#### Visitor SPA — library grid

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Poster-card grid with title, year, IMDb rating badge `[REQ]` | Letterboxd, Plex, Jellyfin, TMDB all default to a poster grid; a text list reads as "spreadsheet", not "catalog" | LOW | 2:3 poster aspect, `w342`/`w500` TMDB sizes, `loading="lazy"`, fixed aspect box to avoid layout shift |
| Placeholder poster when TMDB has none | Hundreds of titles guarantee a few missing posters; broken image icons look abandoned | LOW | Title-text fallback card |
| Sold / unavailable state visible in grid `[REQ: sale state]` | Buyers scanning for something to buy must not fall in love with a sold title; every for-sale catalog (eBay, Discogs) greys or badges sold items | LOW | Desaturate + "SOLD" ribbon; default filter hides sold, toggle shows them |
| Price and condition on card or on hover/detail `[REQ: optional price/condition]` | A "for sale" catalog without a price forces a contact just to ask; eBay/Discogs surface price in the grid | LOW | Show "Ask" when price is null; condition as short chip (Like New / Very Good / Good / Acceptable — eBay's vocabulary) |
| Responsive, phone-first layout `[REQ]` | Buyers browse on the go; PROJECT.md says phone-first | MEDIUM | 2 columns at 360px, 3–4 at tablet, 5–6 at desktop; touch targets ≥44px |
| Hover/tap affordance | Letterboxd shows title/year on poster hover; on touch the card tap opens detail | LOW | Do not hide essential info behind hover only (mobile has no hover) |
| Result count ("Showing 42 of 318") | Every filterable catalog shows it; confirms filters applied | LOW | |
| Empty-state message when filters match nothing | Otherwise the grid looks broken | LOW | "No matches — clear filters" button |

#### Visitor SPA — filter / sort / search

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Filter by genre `[REQ]` | Letterboxd, Plex, Jellyfin, TMDB all have it; primary way buyers narrow | LOW | Multi-select (OR within genre). TMDB returns genre IDs; bake genre names into JSON |
| Sort by popularity `[REQ]` | Letterboxd's default sort; surfaces well-known titles first | LOW | TMDB `popularity` snapshot at ingest time — it drifts, note "as of" in data |
| Sort/filter by IMDb rating `[REQ]` | IMDb rating is the number buyers trust; Letterboxd filters by rating range | LOW | Range slider or preset chips (7+, 8+). Needs rating source decision (TMDB `vote_average` vs true IMDb via OMDb — see STACK.md) |
| Title search `[REQ]` | Non-negotiable for 300+ items | LOW | Client-side substring + diacritics-insensitive; instant, no submit button |
| Sort by title (A–Z) and year | Standard secondary sorts everywhere | LOW | |
| Filter by decade / year | Letterboxd's most-used filter after genre; DVD buyers are often nostalgic browsers | LOW | Decade chips derived from `release_date` |
| Availability filter (Available / Sold / All) | For-sale catalogs default to "in stock" | LOW | Default = Available only |
| Filter state persisted in URL | Shareable "here's all the 90s horror" links; back button restores; static-hosting friendly | MEDIUM | Query string (`?genre=horror&decade=1990&sort=rating`) — works on GitHub Pages without rewrites. nuqs or hand-rolled `URLSearchParams`. Hash-routing not needed if each movie has its own exported page |
| Clear-all filters | Every filter UI has it | LOW | |
| Filters and sorts compose | Buyers expect "horror AND 8+ AND available, sorted by year" | LOW | Pure client-side array pipeline over the JSON |

#### Visitor SPA — detail page

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Shareable deep-link URL per movie `[REQ]` | Seller texts a buyer a link; buyer shares with a friend | MEDIUM | Static export of `/movie/[slug]/` from the JSON at build time (`generateStaticParams`) — avoids 404-fallback hacks. Slug = `title-year-tmdbid` for uniqueness |
| Poster + backdrop hero | Plex preplay layout; visual anchor | LOW | TMDB `backdrop_path` at `w1280`; gradient overlay for text legibility |
| Title, year, runtime, genres, certification | Plex/IMDb header badges; certification (PG-13, R) matters to parents buying discs | LOW | Certification from TMDB `release_dates` (US) — one extra `append_to_response` item |
| Tagline + synopsis `[REQ]` | Plex/IMDb | LOW | TMDB `tagline`, `overview` |
| Director (and writer) `[REQ]` | Plex/IMDb | LOW | From `credits.crew` filtered by job |
| Top-billed cast with headshots and character names `[REQ]` | Plex cast row; IMDb top cast | MEDIUM | Cap at 8–12 from `credits.cast` ordered by `order`; TMDB `profile_path` at `w185`; headshot-missing fallback (initials) |
| IMDb rating with link out to IMDb `[REQ]` | Buyer trust anchor; PROJECT.md requires the link | LOW | `https://www.imdb.com/title/{imdb_id}/` from `external_ids`. Never scrape IMDb |
| Sale block: price, condition, format, availability | This is the entire point of the site; eBay/Discogs put it above the fold | LOW | Sticky on mobile |
| Contact / "How to buy" CTA with no backend | Zero-hosting rules out checkout; buyers still need a one-tap path | LOW | `mailto:` with prefilled subject `Interested in: {Title} ({Year})` and body including the deep link; optionally alternate channel link (SMS, WhatsApp, Signal). Seller contact is config, not hard-coded |
| Sold state on detail page | Deep links outlive the sale | LOW | Banner + disable CTA, keep page (don't 404 sold items) |
| Back to grid preserving filters | Users hate losing their filter set | LOW | Falls out of URL-persisted filter state + `history.back()` or link with preserved query |

#### Visitor SPA — collection level

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Total count and available count in header | Libib public pages and every store show "N items" | LOW | |
| Seller intro / how-it-works blurb | Buyers need to know this is a private sale, pickup/shipping terms, currency | LOW | Static config markdown |

#### Ingestion skill

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Accept one or many photos `[REQ]` | A shelf is dozens of discs; per-disc photos are a non-starter | LOW | Glob/dir input |
| Read multiple spines per photo, rotated 90° either way `[REQ]` | Spines are vertical; a shelf photo has 15–40 spines | HIGH | Vision model reads whole photo; rotate-and-reread when confidence low. Model-side, not code-side, but prompt design is the work |
| Disambiguate film by year/edition `[REQ]` | "The Thing", "Halloween", "Dune" each have multiple films; wrong pick = wrong poster and rating | MEDIUM | TMDB `/search/movie` with `year`/`primary_release_year` when the spine shows a year or studio; else present top candidates ranked by `popularity` and ask |
| Look up metadata via TMDB `[REQ]` | Seller never hand-enters | LOW | One `/movie/{id}?append_to_response=credits,external_ids,videos,release_dates,similar` call per title |
| Append validated entries to JSON `[REQ]` | The SPA reads only this file | LOW | Schema-validate (zod) before write; stable key order for clean git diffs |
| Dedupe against existing entries `[REQ]` | Re-shooting a shelf must not create duplicates | LOW | Key on `tmdb_id`; on hit, offer "increment quantity" or "skip" |
| Flag low-confidence reads instead of guessing `[REQ]` | A silently wrong title is worse than a missing one — it ships a wrong product to a buyer | MEDIUM | Per-item confidence (HIGH/MEDIUM/LOW) from vision read + TMDB match score; LOW items go to a review list |
| Preview / dry-run before commit | Every batch tool (tmm's scrape dialog, Libib import) previews matches first | MEDIUM | Print table: spine text → matched title (year) → confidence; seller approves/edits/rejects |
| Batch throttling of API calls | Free-tier courtesy; TMDB tolerates ~40–50 req/s but bursts from a batch of 300 look like abuse | LOW | Simple concurrency limit (e.g. 4) + retry on 429 |
| Reusable by other sellers `[REQ]` | Skill is a product too | LOW | API key via env, catalog path and seller contact via config file; no Patrick-specific constants |
| Mark an entry sold / edit an entry | Selling is the point; entries change | LOW | Skill subcommand or documented manual JSON edit; a `sold_at` date. Without this the seller edits JSON by hand and breaks the schema |

### Differentiators (Competitive Advantage)

Not required, but these are why a buyer bookmarks this over an eBay lot listing and why a seller picks this skill over Libib.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Actor / director search | "Anything with Harrison Ford?" — Letterboxd/Plex do this; eBay lots cannot | LOW once cast is ingested | Search index over `cast[].name` + `director`; show "matched via: Cast" hint. **Depends on cast ingestion** |
| "More in this collection" (similar titles you can also buy) | Cross-sell; Plex "Related" row but scoped to what's actually for sale | MEDIUM | Intersect TMDB `similar` IDs with catalog, fall back to shared genre + decade. Ranked at build time |
| Trailer link (YouTube, no embed) | Plex extras row; helps buyers decide; zero cost as a link | LOW | TMDB `videos` → first `type: Trailer, site: YouTube`; link out, never iframe (privacy, weight) |
| TMDB link + Letterboxd link alongside IMDb | Film buffs use Letterboxd; costs nothing | LOW | Letterboxd resolves `https://letterboxd.com/tmdb/{id}` |
| Bundle / box-set awareness | Trilogy box sets and complete-series sets are the highest-value DVD items; buyers filter for them on eBay | MEDIUM | Entry type `set` with child `tmdb_id`s (TMDB `belongs_to_collection` helps); grid shows set as one card with "3 films" badge |
| Format + region badge (DVD / Blu-ray / 4K; Region 1/2/free) | eBay's top buyer filters; international buyers must know region | LOW | Ingestion asks once per batch ("this shelf is Region 1 DVD?") and applies default |
| Edition disambiguation surfaced in UI (Director's Cut, Criterion, Extended, Steelbook) | Collectors pay for these; eBay has it as an item-specific | LOW UI / MEDIUM ingestion | Free-text `edition` field; skill reads it from spine when printed |
| Recently added row / "New this week" | Repeat buyers see what changed; Plex home row pattern | LOW | Sort by `added_at` desc, top 12 |
| Collection stats (by genre, by decade) | Makes the catalog feel curated; fun to share | LOW | Static counts computed at build; simple bars, no chart library |
| Density toggle (compact / comfortable) | Letterboxd-style power browsing on desktop; trivial with a grid class swap | LOW | Persist in `localStorage` |
| Keyboard-friendly search (`/` focuses search) | Delight for desktop | LOW | |
| Open Graph / Twitter card per movie page | Shared links in iMessage/WhatsApp show poster + title + price — this is the seller's main marketing surface | LOW | Static `<meta>` per exported page using TMDB poster URL |
| Wishlist / "hold for me" via prefilled mailto listing multiple titles | Buyers pick 5 discs and send one message; approximates a cart with zero backend | MEDIUM | `localStorage` selection + `mailto:` body enumerating titles and deep links. Cap message length (~2000 chars for mailto safety) |
| Local image cache mode `[REQ]` | Fully self-contained site if TMDB CDN ever changes terms | MEDIUM | Build flag downloads posters/headshots to `public/img/`; default off |
| Ingestion: quantity per title | Two copies of the same DVD is common | LOW | `quantity` field; dedupe hit increments |
| Ingestion: per-batch defaults (condition, price, format, region) | Shelf-level attributes are uniform; asking per disc is tedious | LOW | Batch flags with per-item override in preview |
| Ingestion: rotate-and-reread fallback | Recovers spines whose text direction confused the first pass | MEDIUM | Second vision pass on the rotated image only for items missing/low-confidence |
| Ingestion: `--refresh` metadata for existing entries | Ratings and popularity drift; posters get replaced | LOW | Re-fetch by `tmdb_id`, preserve sale fields |
| Ingestion: git-friendly output | Diff shows exactly what was added; history is the audit log | LOW | Deterministic key order, one entry per commit chunk, `added_at` |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Cart / checkout / payments | "It's a store" | Needs a backend or Stripe/Shopify integration and inventory sync; contradicts zero-hosting; PROJECT.md out of scope | Multi-title prefilled `mailto:` "inquiry list"; seller invoices via PayPal/e-transfer out of band |
| User accounts, login, saved lists | Letterboxd/Libib have them | No backend; nothing to log in to | `localStorage` wishlist only, no sync |
| Comments, reviews, user ratings | IMDb/Letterboxd have them | No backend; spam surface; buyer doesn't need them to buy a disc | Show IMDb rating + link to IMDb/Letterboxd reviews |
| Live TMDB/OMDb API calls from the visitor's browser | "Always-fresh ratings" | Exposes API key in client bundle; rate-limits the seller's key; site breaks when key rotates; PROJECT.md constraint forbids | Bake metadata at ingest; `--refresh` subcommand for periodic updates |
| Scraping imdb.com for posters/ratings | IMDb is what buyers know | ToS violation, blocks, breaks silently | TMDB for images, `imdb_id` for link-out; OMDb for true IMDb rating if wanted |
| Storing all poster images in the repo by default | Self-contained, no external dependency | 300 posters × 100–300 KB bloats the repo and Pages artifact; headshots multiply it | TMDB CDN by default; opt-in local cache mode |
| Embedded YouTube trailer iframes | Plex shows trailers | Third-party cookies, 500 KB+ per page, CSP headaches, autoplay annoyances | Plain link to YouTube with a play-icon thumbnail |
| Playback / media-server features | "Plex clone" | Explicitly out of scope; catalog only | Borrow the preplay UX only |
| Server-side full-text search (Algolia, etc.) | "Search should be fast" | Adds a service; 300 items fits in memory trivially | Client-side filter over the JSON; optional Fuse.js for fuzzy |
| Pagination / infinite scroll | Big-catalog pattern | 300 poster cards with lazy images render fine; pagination hides items behind clicks and breaks "filter down in three clicks" | Single page, lazy-loaded images, optional "show more" beyond 200 if ever needed |
| Real-time inventory / "reserved" status | Avoid double-selling | Needs writes from the web; no backend | Seller marks sold via skill and redeploys (minutes); CTA text says "subject to availability" |
| Automatic pricing from eBay comps | Seller convenience | eBay API access is gated; scraping is brittle; prices vary by condition | Batch default price + manual override; seller research is out of band |
| Per-disc photos of actual item | eBay buyers expect real photos | Contradicts "no hosted images"; multiplies repo size; DVDs are commodity items | Condition chip + honest notes field; offer photos on request in CTA |
| Fully automatic ingestion with no review step | "Just do it" | Vision misreads (remakes, similar titles, sequels numbered on spine) ship wrong products | Dry-run preview with confidence flags is table stakes |
| Analytics / tracking scripts | Seller curiosity | Cookie banners, weight, privacy; unlisted page has tiny traffic | GitHub Pages traffic insights, or nothing |

## Feature Dependencies

```
[Catalog JSON schema]
    └──required by──> [Poster grid]
    └──required by──> [Detail page static export]
    └──required by──> [All filters/sorts]
    └──required by──> [Ingestion skill write path]

[Ingestion: TMDB lookup with credits]
    └──enables──> [Detail: cast with headshots]
                      └──enables──> [Actor/director search]
    └──enables──> [Detail: director]
[Ingestion: append external_ids]
    └──enables──> [IMDb link-out]
[Ingestion: append videos]
    └──enables──> [Trailer link]
[Ingestion: append release_dates]
    └──enables──> [Certification badge]
[Ingestion: append similar + belongs_to_collection]
    └──enables──> [More in this collection]
    └──enables──> [Box-set awareness]

[Sale fields (price, condition, status, format, region)]
    └──required by──> [Sold overlay]
    └──required by──> [Availability filter]
    └──required by──> [Sale block + contact CTA]
    └──required by──> [Format/region badge]

[URL-persisted filter state] ──enhances──> [Back-to-grid preserving filters]
[URL-persisted filter state] ──enhances──> [Shareable filtered links]

[Detail page static export] ──required by──> [Open Graph per movie]
[Detail page static export] ──required by──> [Shareable deep link]
[Detail page static export] ──required by──> [mailto with deep link]

[Vision read] ──> [Disambiguation] ──> [TMDB lookup] ──> [Dedupe] ──> [Preview/dry-run] ──> [JSON write]
[Confidence flags] ──enhances──> [Preview/dry-run]
[Rotate-and-reread] ──enhances──> [Vision read]

[Wishlist mailto] ──requires──> [localStorage selection] + [Detail deep link]

[Local image cache mode] ──conflicts──> [Default CDN loading]   (mutually exclusive build modes, not both)
[Live client-side API calls] ──conflicts──> [Baked metadata]    (anti-feature; choose baked)
```

### Dependency Notes

- **Actor search requires cast ingestion:** searching by actor is a trivial client-side index *only if* the skill stores `cast[].name` (top 10–12) in the JSON. Decide in the schema phase; retrofitting means re-fetching every title.
- **Detail page fields require `append_to_response` at ingest:** credits, external_ids, videos, release_dates, similar are one TMDB call per title. Fetch all of them on first ingest even if the UI ships without trailer/certification — the marginal cost is zero and it avoids a `--refresh` pass later.
- **Every sale-related UI feature depends on the sale schema:** `status` (`available` | `sold` | `reserved`?), `price`, `currency`, `condition`, `format`, `region`, `edition`, `quantity`, `notes`, `added_at`, `sold_at`. Lock this before the grid is built.
- **Deep links depend on static export of per-movie routes:** `generateStaticParams` over the JSON at build time. This is also what makes Open Graph previews and `mailto` bodies work.
- **URL filter state is what makes "back to grid" and shareable searches work** — build it into the grid from day one rather than adding later.
- **Preview/dry-run gates JSON write:** the skill must never write without either an approval step or an explicit `--yes` flag.
- **Local image cache conflicts with CDN default:** one build flag; never mixed within a build.

## MVP Definition

### Launch With (v1)

- [ ] Catalog JSON schema with metadata + sale fields — everything reads from it
- [ ] Ingestion skill: photos → vision read → TMDB disambiguation → full metadata (credits, external_ids, videos, release_dates, similar) → dedupe → preview with confidence flags → append — the seller cannot populate 300 titles otherwise
- [ ] Ingestion: batch defaults (condition, price, format, region) — makes the preview step fast
- [ ] Ingestion: mark-sold / edit subcommand — seller will need it within days of launch
- [ ] Poster grid with title/year/rating/price/sold state, placeholder posters, lazy loading, responsive
- [ ] Filters: genre (multi), decade, rating threshold, availability; sorts: popularity, rating, year, title; title search; URL-persisted; result count; clear-all
- [ ] Detail page (static-exported, deep-linkable): backdrop, poster, badges, tagline, synopsis, director, top cast with headshots, IMDb link, sale block, contact CTA (mailto prefilled), sold banner
- [ ] Header: count, seller blurb, how-to-buy terms
- [ ] Open Graph meta per movie — shared links are the distribution channel for an unlisted site

### Add After Validation (v1.x)

- [ ] Actor/director search — once cast data exists it's a small change; add when a buyer asks "anything with X?"
- [ ] More-in-this-collection row — when catalog exceeds ~100 so intersections are non-empty
- [ ] Trailer + TMDB + Letterboxd links — one-liners once `videos` is in the data
- [ ] Recently added row — once ingestion runs in multiple batches
- [ ] Wishlist mailto (multi-title inquiry) — when buyers start asking for several titles per message
- [ ] Box-set entry type — when the first trilogy/series set is ingested
- [ ] Density toggle, `/` shortcut — desktop polish
- [ ] Ingestion `--refresh` — when ratings look stale

### Future Consideration (v2+)

- [ ] Collection stats page — fun, not load-bearing
- [ ] Local image cache mode `[REQ, opt-in]` — build it when there's a reason (TMDB terms change, offline demo); keep the flag reserved in config from v1
- [ ] Rotate-and-reread vision fallback — only if v1 misread rate is high in practice
- [ ] Skill packaging for other sellers (README, config template, example photos) — after Patrick's own catalog validates the flow

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Catalog JSON schema (metadata + sale fields) | HIGH | LOW | P1 |
| Ingestion: vision read + TMDB disambiguation + preview | HIGH | HIGH | P1 |
| Ingestion: dedupe + confidence flags | HIGH | MEDIUM | P1 |
| Ingestion: batch defaults, mark-sold/edit | HIGH | LOW | P1 |
| Poster grid + sold overlay + price chip | HIGH | LOW | P1 |
| Genre / decade / rating / availability filters + sorts | HIGH | LOW | P1 |
| Title search | HIGH | LOW | P1 |
| URL-persisted filter state | MEDIUM | MEDIUM | P1 |
| Static-exported detail page with cast + IMDb link | HIGH | MEDIUM | P1 |
| Contact CTA (prefilled mailto) | HIGH | LOW | P1 |
| Open Graph per movie | HIGH | LOW | P1 |
| Responsive phone-first layout | HIGH | MEDIUM | P1 |
| Actor/director search | MEDIUM | LOW | P2 |
| More in this collection | MEDIUM | MEDIUM | P2 |
| Trailer / TMDB / Letterboxd links | MEDIUM | LOW | P2 |
| Format / region / edition badges | MEDIUM | LOW | P2 |
| Recently added row | MEDIUM | LOW | P2 |
| Wishlist mailto | MEDIUM | MEDIUM | P2 |
| Box-set entries | MEDIUM | MEDIUM | P2 |
| Ingestion `--refresh` | LOW | LOW | P2 |
| Density toggle, keyboard shortcuts | LOW | LOW | P3 |
| Collection stats | LOW | LOW | P3 |
| Local image cache mode | LOW | MEDIUM | P3 |
| Rotate-and-reread fallback | LOW | MEDIUM | P3 |

**Priority key:** P1 must have for launch · P2 should have, add when possible · P3 nice to have, future consideration

## Competitor Feature Analysis

| Feature | Letterboxd | Plex / Jellyfin | Libib (published) | eBay / Discogs listings | Our Approach |
|---------|-----------|-----------------|-------------------|-------------------------|--------------|
| Browse view | Poster grid, hover shows title/year | Poster grid, density options | Cover grid or list | Photo thumbnails, list | Poster grid, title/year/rating/price always visible (no hover-only info) |
| Filter | Decade, genre, rating (single-select each), streaming service | Genre, year, rating, unwatched, resolution | Tags, search (Pro) | Item specifics: format, region, edition, rating, condition | Genre (multi), decade, rating threshold, availability, format; all URL-persisted |
| Sort | Popularity, name, release, rating, length | Title, year, rating, date added | Title, author, date added | Price, ending soonest, best match | Popularity (default), rating, year, title, recently added |
| Search | Title, people | Title, people, everything | Title (Pro) | Keyword | Title (v1), actor/director (v1.x) |
| Detail page | Poster, backdrop, synopsis, cast, crew, ratings histogram, reviews, where-to-watch | Backdrop hero, badges, synopsis, cast row, extras (trailers), related row | Cover, synopsis, notes, availability | Photos, item specifics, condition notes, price, buy button | Plex layout + eBay sale block; IMDb/TMDB/trailer link-outs; no reviews/histogram |
| Sale / availability | n/a | n/a | "Available / checked out" (Pro) | Price, condition, sold state, quantity | Price, condition, format/region/edition, sold overlay, quantity |
| Buy path | n/a | n/a | Place hold (Pro) | Cart/checkout | Prefilled mailto / seller channel; optional multi-title inquiry |
| Ingestion | Manual log | Filename scan → agent match; manual "Fix match" dialog | Barcode scan, ISBN/UPC lookup, CSV import | Manual form, catalog lookup by UPC | Spine photo → vision → TMDB match → preview/fix → JSON; no barcodes needed (spines are what's visible on a shelf) |
| Duplicate handling | n/a | Merge/split | Flags duplicates | n/a | Dedupe on `tmdb_id`, increments quantity |
| Wrong-match recovery | n/a | "Fix incorrect match" search dialog | Edit item | Edit listing | Preview shows candidates; `edit` subcommand re-searches by title/year |
| Hosting | SaaS | Self-hosted server | SaaS | Marketplace | Static files on GitHub Pages, zero cost |

## Ingestion-Specific Findings

- **Whole-photo vision read is viable; verification against TMDB is mandatory.** Bookshelf-spine projects that moved from OCR to VLMs (GPT-4V, Gemini, Moondream2) report good raw reads, but every mature pipeline still matches the read text against a catalog DB with fuzzy/candidate scoring, because partial occlusion, truncated text, and mixed fonts produce plausible-but-wrong strings. Treat the vision read as a *query*, not an answer.
- **Disambiguation signals available on a spine:** studio logo (Criterion, Disney), year (sometimes), edition words ("Director's Cut", "Unrated", "Special Edition"), sequel numerals, format logo (DVD vs Blu-ray band colour — blue case = Blu-ray, is visible in colour photos). Prompt the model to extract these as structured fields, not just a title string.
- **TMDB search flow that avoids most ambiguity:** `/search/movie?query=<title>&year=<y>` when a year is read; otherwise `/search/movie?query=<title>` and auto-accept only when the top result's popularity is ≥3× the second *and* the title matches near-exactly; everything else is MEDIUM/LOW confidence and goes to preview.
- **Box sets:** a spine reading "The Lord of the Rings Trilogy" should produce one `set` entry with three child films (TMDB `belongs_to_collection` gives the collection ID and members). Don't split into three separate for-sale entries — the physical item is one box.
- **Throttling:** TMDB's current published stance is no hard rate limit but ~50 req/s tolerance; a batch of 300 titles at concurrency 4 with 429-retry finishes in under a minute and never trips it. No need for elaborate scheduling.
- **Seller mental model:** the flow that Plex ("Fix match"), tinyMediaManager (scrape → review → save), and Libib (scan → confirm) converge on is *propose, review, commit*. Ship the preview table even if it feels like friction; it is what makes the seller trust the output.

## Sources

Confidence per the classify-confidence seam: individual `websearch`/`webfetch` items = LOW; cross-corroborated synthesis presented here as MEDIUM.

- Letterboxd films browser, sorting and filters — https://letterboxd.com/films/ , https://letterboxd.com/journal/sorting/ , https://www.fivestarinsider.com/sorting-and-filtering-on-letterboxd/
- Plex item details ("pre-play") page, Cast/Related/Extras rows, trailers from metadata provider — https://support.plex.tv/articles/202462186-viewing-item-details/ , https://support.plex.tv/articles/202920803-extras/
- Libib published collections (public URL, search/tags/sort/availability in Pro) — https://support.libib.com/support/published-libraries-public/ , https://www.libib.com/
- eBay DVD item specifics buyers filter on (Format, Region Code, Edition, Rating, Condition) — https://www.ebay.com/sellercenter/listings/item-specifics , https://www.ebay.com/help/selling/listings/creating-managing-listings/item-conditions-category?id=4765
- What DVD buyers value (condition, original case, editions, bundles, complete seasons) — https://useflippr.com/blog/how-to-sell-dvds-15-tips , https://10web.io/blog/how-to-sell-dvds-online/
- tinyMediaManager features (search/sort/filter, movie sets, scrape-and-review) — https://www.tinymediamanager.org/features/
- Spine-photo identification pipelines (segment → read → match) — https://github.com/suxrobgm/bookshelf-scanner , https://github.com/Nisarg851/BookSpinesOCR , https://jamesg.blog/2024/02/22/photo-bookshelf , https://arxiv.org/html/2407.19812v1 , https://apps.apple.com/us/app/shelf-scan-book-spine-scanner/id6754824642
- TMDB `/search/movie` parameters (`query`, `year`, `primary_release_year`, `region`) and result fields — https://developer.themoviedb.org/reference/search-movie
- TMDB `append_to_response` (credits, videos, similar, external_ids, release_dates; up to 20 per call) — https://developer.themoviedb.org/docs/append-to-response
- URL-as-state for filters in Next.js/React (nuqs) — https://nuqs.dev/ , https://github.com/47ng/nuqs
- Project context — /home/patrick/github.com/patrickclery/dvd-seller/.planning/PROJECT.md

---
*Feature research for: static DVD-for-sale catalog SPA + photo ingestion skill*
*Researched: 2026-09-21*
