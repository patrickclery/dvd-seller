# DVD Seller

## What This Is

A static, reactive single-page web app that catalogs a personal DVD collection for sale — an "IMDb clone for selling DVDs." Visitors browse a library of hundreds of titles, filter by genre, popularity, and IMDb rating, and open a Plex-style detail page per movie showing poster, synopsis, cast, and a link out to IMDb. The catalog is populated by an AI agent (packaged as a skill) that reads photos of DVD spines, identifies each title, pulls metadata from a movie database, and appends it to a JSON data file that the SPA reads. Ships as flat files on GitHub Pages with no backend and no hosted image assets.

## Core Value

Turn a stack of DVD spine photos into a browsable, filterable, shareable catalog with zero hosting cost — a buyer can find a movie they want in seconds, and the seller never hand-enters a title.

## Business Context

- **Customer**: Patrick (seller) first; any DVD seller who installs the skill second. Buyers are visitors who know the URL.
- **Revenue model**: Sale of physical DVDs. The site is a catalog, not a checkout.
- **Success metric**: Every DVD in the physical collection appears in the catalog with correct metadata, and a visitor can filter down to a title in under three clicks.
- **Strategy notes**: Initially unlisted — reachable only at `/sale/dvds` and not linked from the patrickclery.com homepage.

## Requirements

### Validated

(None yet — ship to validate)

### Active

**Catalog SPA (visitor-facing)**
- [ ] Visitor sees a grid/list of all DVDs with poster art, title, year, and IMDb rating
- [ ] Visitor can filter by genre/category
- [ ] Visitor can sort/filter by popularity
- [ ] Visitor can sort/filter by IMDb rating
- [ ] Visitor can search by title
- [ ] Visitor can open a per-movie detail page (Plex-style) with poster, synopsis, year, runtime, genres, rating, director, and cast with headshots
- [ ] Detail page links out to the movie's IMDb page
- [ ] Each movie has a shareable, deep-linkable URL that works on static hosting
- [ ] Layout is responsive (phone-first for buyers browsing on the go)

**Data & assets**
- [ ] Catalog is a committed JSON (or similar) data file the SPA reads at build/load time — no database server
- [ ] Poster and headshot images are loaded on demand from an external metadata provider CDN (not stored in the repo) by default
- [ ] Optional local image caching mode exists so the site can be fully self-contained if desired
- [ ] Image loading never triggers rate limiting from IMDb or the metadata provider (use a provider with a real API and image CDN; avoid scraping imdb.com)
- [ ] Each entry records sale state (available / sold) and optional price/condition

**Ingestion skill (seller-facing, agent-driven)**
- [ ] A Claude Code skill accepts one or many photos of DVD spines
- [ ] The skill reads the titles from the spines (vision), disambiguates the correct film (year/edition), and looks up metadata via an API
- [ ] The skill appends validated entries to the catalog data file, deduplicating against existing entries
- [ ] The skill flags low-confidence reads for the seller to confirm rather than guessing silently
- [ ] The skill is reusable by any seller who installs it — no hard-coded Patrick-specific paths or keys

**Hosting & deployment**
- [ ] Lives in its own GitHub repo (`patrickclery/dvd-seller`), created via `gh`
- [ ] Builds locally to static files and deploys to GitHub Pages the same way patrickclery.github.io does (Actions workflow, `out/` artifact)
- [ ] Built with a configurable base path so it can be served at `patrickclery.com/sale/dvds`
- [ ] Not linked from the patrickclery.com homepage; discoverable only by URL
- [ ] Zero required paid hosting — GitHub Pages plus free-tier API is the whole stack

### Out of Scope

- Checkout / payments / cart — the site is a catalog; buyers contact the seller. Adding commerce means a backend or third-party, which violates zero-hosting.
- User accounts, logins, reviews, comments — no backend to store them.
- Scraping imdb.com HTML for posters or data — violates IMDb terms and will get rate-limited/blocked; use a licensed API (TMDB/OMDb) that provides IMDb IDs and ratings.
- Server-side rendering or API routes — GitHub Pages serves flat files only.
- Storing hundreds of poster images in the repo by default — bloats the repo and GitHub Pages; on-demand CDN loading is the default, local cache is opt-in.
- Building a general-purpose media server (Plex replacement) — only the browsing/detail UX is borrowed, not playback or library management.
- Mobile native app — responsive web is sufficient.

## Context

- **Deployment sibling**: `patrickclery.github.io` (local clone at `~/github.com/patrickclery/patrickclery.github.io`) is a Next.js 15 App Router site with `output: "export"`, `trailingSlash: true`, `images.unoptimized: true`, Tailwind v4, TypeScript strict, lucide-react icons, deployed by `.github/workflows/deploy.yml` (Node 20, `npm ci && npm run build`, upload `out/`, `actions/deploy-pages@v4`). Custom domain via `public/CNAME` = `patrickclery.com`. Conventions there: PascalCase `.tsx` components with `export function`, inline prop types, 2-space indent, no linter config.
- **Repo separation**: The user explicitly wants this in its own repo, NOT inside patrickclery.github.io, but deployed the same way. Reaching `patrickclery.com/sale/dvds` from a separate repo is a routing question — GitHub project Pages serve at `patrickclery.github.io/dvd-seller/` by default. Two viable paths: (a) main-site build pulls this repo's `out/` into `out/sale/dvds/`; (b) serve on a subdomain via CNAME. Build with `basePath` configurable so either works. Decision deferred to Phase 1 planning.
- **Metadata source**: IMDb has no free public API. TMDB (The Movie Database) has a free API with poster/headshot CDN (`image.tmdb.org`), genres, cast/crew, popularity score, and `imdb_id` for linking out. OMDb exposes IMDb ratings directly (free tier 1k req/day). Research phase will confirm the best combination and rate limits. Images served from TMDB's CDN are loaded by the visitor's browser directly — no requests from our origin, no rate-limit exposure for the site.
- **Ingestion input**: Photos of DVD spines (many DVDs per photo, text rotated 90°, mixed fonts). Vision-capable model reads titles; ambiguity (remakes, editions, box sets) must be surfaced, not guessed.
- **Prior art check requested**: User asked for a quick check of popular, maintained GitHub projects that already do this (personal media catalog SPA with IMDb/TMDB metadata, static hosting) before building from scratch. The Stack researcher is tasked with this; findings go in `research/STACK.md` and SUMMARY.md.
- **Scale**: Hundreds of titles (not thousands). A single JSON file of a few hundred entries is well within static-site limits; no pagination backend needed.

## Constraints

- **Hosting**: GitHub Pages static files only — no SSR, no serverless, no database. Everything the visitor sees must be prebuilt or fetched client-side from public CDNs.
- **Cost**: Zero required paid services. Metadata API must have a free tier adequate for a few hundred lookups during ingestion.
- **Rate limiting**: Must not hammer IMDb or the metadata provider. Ingestion is batched and throttled; the live site makes no metadata API calls (data is baked in at build time); images come from the provider's CDN.
- **Tech stack**: Match patrickclery.github.io (Next.js static export, React 19, TypeScript, Tailwind v4) so the two repos share conventions and can be composed. Base path must be configurable via env var.
- **Repo**: Standalone `patrickclery/dvd-seller` on GitHub, created with `gh`. Deploy workflow mirrors the sibling repo.
- **Discoverability**: Not on the main homepage. No `noindex` requirement stated — just unlinked.
- **Reusability**: The ingestion skill must work for other sellers; API keys via env, paths via config.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Standalone repo, deployed like patrickclery.github.io | User's explicit direction; keeps the portfolio repo clean while reusing a proven deploy path | — Pending |
| Next.js static export + React + Tailwind v4 | Match sibling site conventions; already proven on GitHub Pages | — Pending |
| Metadata from a licensed API (TMDB/OMDb), link out to IMDb, never scrape IMDb | Free tier, has IMDb IDs and ratings, image CDN handles poster load; avoids ToS and rate-limit risk | — Pending |
| Images loaded on demand from provider CDN by default; local cache opt-in | Zero hosting for assets; repo stays small; user explicitly OK with on-demand loading | — Pending |
| Catalog stored as committed JSON, populated by an agent skill | No database server; git history is the audit log; skill is portable to other sellers | — Pending |
| Configurable `basePath` (`/sale/dvds`) | Lets the same build serve as its own Pages site or be mounted under patrickclery.com | — Pending |
| Check existing OSS before building | User asked for a quick prior-art check of popular, maintained GitHub projects | — Pending (research) |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-09-21 after initialization*
