<!-- GSD:project-start source:PROJECT.md -->

## Project

**DVD Seller**

A static, reactive single-page web app that catalogs a personal DVD collection for sale — an "IMDb clone for selling DVDs." Visitors browse a library of hundreds of titles, filter by genre, popularity, and IMDb rating, and open a Plex-style detail page per movie showing poster, synopsis, cast, and a link out to IMDb. The catalog is populated by an AI agent (packaged as a skill) that reads photos of DVD spines, identifies each title, pulls metadata from a movie database, and appends it to a JSON data file that the SPA reads. Ships as flat files on GitHub Pages with no backend and no hosted image assets.

**Core Value:** Turn a stack of DVD spine photos into a browsable, filterable, shareable catalog with zero hosting cost — a buyer can find a movie they want in seconds, and the seller never hand-enters a title.

### Constraints

- **Hosting**: GitHub Pages static files only — no SSR, no serverless, no database. Everything the visitor sees must be prebuilt or fetched client-side from public CDNs.
- **Cost**: Zero required paid services. Metadata API must have a free tier adequate for a few hundred lookups during ingestion.
- **Rate limiting**: Must not hammer IMDb or the metadata provider. Ingestion is batched and throttled; the live site makes no metadata API calls (data is baked in at build time); images come from the provider's CDN.
- **Tech stack**: Match patrickclery.github.io (Next.js static export, React 19, TypeScript, Tailwind v4) so the two repos share conventions and can be composed. Base path must be configurable via env var.
- **Repo**: Standalone `patrickclery/dvd-seller` on GitHub, created with `gh`. Deploy workflow mirrors the sibling repo.
- **Discoverability**: Not on the main homepage. No `noindex` requirement stated — just unlinked.
- **Reusability**: The ingestion skill must work for other sellers; API keys via env, paths via config.

<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->

## Technology Stack

## Prior Art: Existing Open-Source Projects

| Project | Stars | Last push | License | Stack | Flat static on GH Pages? | Import from photos/OCR? | For-sale / sold state? | Verdict |
|---|---|---|---|---|---|---|---|---|
| [Kyonew/DVinyl](https://github.com/Kyonew/DVinyl) | 224 | 2026-09-16 | MIT | Node + EJS, Docker, TMDB/Discogs/IGDB plugins, DB | **No** (server + DB; Docker) | Barcode scanner only; manual/ID entry | Has "market value" for music; no sale state for movies | Closest conceptual match (physical media incl. DVD/Blu-ray), but needs a server |
| [hacan359/tonkatsu_box](https://github.com/hacan359/tonkatsu_box) | 551 | 2026-09-16 | MIT | Flutter/Dart desktop+mobile app, optional self-hosted web | **No** (app; web mode needs hosting) | No | No | Personal tracker, not a public catalog |
| [leepeuker/movary](https://github.com/leepeuker/movary) | 779 | 2026-09-08 | MIT | PHP + MySQL/SQLite, Docker | **No** | No | No (watch history/ratings) | Watch tracker, not a collection-for-sale site |
| [FuzzyGrim/Yamtrack](https://github.com/FuzzyGrim/Yamtrack) | 3,618 | 2026-09-18 | AGPL-3.0 | Python/Django, Docker | **No** | No | No | Media tracker |
| [IgnisDa/ryot](https://github.com/IgnisDa/ryot) | 3,604 | 2026-09-22 | GPL-3.0 | Rust + React, Postgres | **No** | No | No | Media tracker |
| [sbondCo/Watcharr](https://github.com/sbondCo/Watcharr) | 1,508 | 2026-09-16 | GPL-3.0 | Go + Svelte, SQLite | **No** | No | No | Watched list |
| [bonukai/MediaTracker](https://github.com/bonukai/MediaTracker) | 934 | 2025-02-20 | MIT | TypeScript/Node, SQLite | **No** | No | No | Stale ~19 months |
| [devfake/flox](https://github.com/devfake/flox) | 1,351 | 2026-07-05 | MIT | PHP/Laravel + Vue | **No** | No | No | **Archived** |
| [jreklund/php4dvd](https://github.com/jreklund/php4dvd) | 86 | 2025-12-22 | GPL-3.0 | PHP + MySQL | **No** | No | Tracks "bought / loaned out", not for-sale | Only repo with explicit DVD-collection + "own/loaned" semantics; scrapes IMDb (ToS risk) |
| [seerr-team/seerr](https://github.com/seerr-team/seerr) (Overseerr successor) | 12,652 | 2026-09-22 | MIT | Next.js + Node, SQLite/Postgres | **No** | No | No | Request manager for Plex/Jellyfin; UI is Plex-like but tightly coupled to server |
| Radarr / Jellyfin / Plex | 14k / 57k / n/a | active | GPL / GPL / proprietary | C# servers | **No** | No | No | Media servers/PVRs, explicitly out of scope |
| tinyMediaManager | GitLab (GitHub mirror 8 stars, 2022) | — | proprietary-ish | Java desktop | **No** (desktop; exports NFO/HTML) | No | No | Desktop scraper, not a site |
| [Superschnizel/Obsidian-Moviegrabber](https://github.com/Superschnizel/Obsidian-Moviegrabber) | 35 | 2026-02-10 | MIT | Obsidian plugin | **No** (notes, not a site) | No | No | Obsidian only |
| [justlep/dvd-archive](https://github.com/justlep/dvd-archive) | 5 | 2025-03-08 | MIT | Node + Svelte PoC | No | No | No | Toy PoC |
| Static-site attempts (`jsonmc/jsonmc.github.io` "JSON Movie Collection website", `AndrewFraser95/BluRayCollection`, misc "movie-collection-website") | 0-5 | 2019-2025 | none | HTML/JS | Yes | No | No | Zero stars, unlicensed, abandoned |
| Libib, Notion templates | proprietary SaaS / not on GitHub | — | — | — | No (hosted) | Libib: barcode scan | Libib: lending, no sale state | Not open source, not static |
| Hugo/Astro/Eleventy "movie collection" themes | none found with >10 stars | — | — | — | — | — | — | Searches for `hugo movie theme`, `astro movies tmdb`, `eleventy movies` returned no maintained theme |
| Letterboxd clones (janaiscoding/letterboxd-clone 51 stars, etc.) | <60 | 2023-2025 | mostly unlicensed | React + Firebase/MERN | No (Firebase/Mongo) | No | No | Student projects; require backend |

## Metadata & Image Provider

| Capability | TMDB API v3 | OMDb API | IMDb official (2026) |
|---|---|---|---|
| Cost / free tier | Free developer key (account + agree to ToS). No daily cap. | Free key: **1,000 req/day**. Patreon from ~$1-1.50/mo raises quota and unlocks Poster API. | `developer.imdb.com` now redirects to `data.imdb.com`. GraphQL API + datasets sold **only via AWS Data Exchange, contact-sales pricing, no free tier**. Separate non-commercial TSV datasets (`datasets.imdbws.com`, daily refresh, personal/non-commercial only, no images). |
| Rate limit | ~40 req/s soft cap, returns 429; legacy 40/10s removed 2019. | 1,000/day hard. | Datasets: bulk download, no API calls. |
| Returns `imdb_id` | Yes (`/movie/{id}?append_to_response=external_ids` or in movie details) | Yes (`imdbID`) | Yes (tconst) |
| IMDb rating | **No** (`vote_average` is TMDB's own rating) | **Yes** (`imdbRating`, `imdbVotes`) | Yes (`title.ratings.tsv.gz`) |
| Popularity | Yes (`popularity`, TMDB-computed) | No | No (Meters add-on is paid) |
| Genres | Yes (IDs + names) | Yes (comma string) | Yes (basics) |
| Cast with headshots | **Yes** (`credits.cast[].profile_path`) | Actors as comma string, no images | Principals, no images |
| Poster URLs | Yes (`poster_path`, `backdrop_path`) | Amazon-hosted `Poster` URL on free tier; Poster API patrons-only | None |
| Image CDN hotlink policy | Documented usage pattern is browser-side URL construction `https://image.tmdb.org/t/p/{w185,w342,w500,original}/{path}`; ToS prohibits using TMDB "as an image hosting service for banner advertisements, graphics, etc." (i.e. non-movie assets), not poster display. Data may be cached max 6 months. | Poster URLs point at Amazon/IMDb media hosts; hotlinking those is not a licensed use. | n/a |
| Commercial-use terms | Free key is **non-commercial only**; staff (Travis Bell) on TMDB Talk: "if you are not monetizing our content in any way then you are fine". Ads/paid features -> $149/mo commercial tier. | Data CC BY-NC 4.0 (non-commercial). | Non-commercial datasets: personal/non-commercial only. |
| Attribution | Required: TMDB logo (less prominent than own branding, in About/Credits) + text "This product uses the TMDB API but is not endorsed or certified by TMDB." | Attribution per CC BY. | Per license. |

- TMDB is the only provider that supplies everything the UI needs (poster, backdrop, synopsis, runtime, genres, popularity, director, cast *with headshots*, `imdb_id` for the link-out) in one call: `GET /3/movie/{id}?append_to_response=credits,external_ids`. Authenticate with the v4 "API Read Access Token" as `Authorization: Bearer` (works on v3 endpoints, recommended by TMDB docs). Read from `TMDB_API_TOKEN` env var (name locked by Phase 1 D-14).
- Title identification: `GET /3/search/movie?query=<title>&year=<year>` (use `primary_release_year` when the spine shows a year). Ambiguous top-N results are surfaced to the seller by the skill, never auto-picked when the score gap is small.
- IMDb rating: TMDB does not carry it. Use OMDb `GET /?i=<imdb_id>&apikey=<OMDB_KEY>` to fetch `imdbRating`. At ~500 titles this is one afternoon under the 1,000/day free limit. Make it optional: if `OMDB_API_KEY` is unset, the site shows the TMDB rating labelled as such. Store `imdbRating` as a snapshot string with a `fetchedAt` date; it does not need to be live.
- Images: store only the TMDB `poster_path` / `backdrop_path` / `profile_path` fragments in JSON. The browser builds `https://image.tmdb.org/t/p/w342${poster_path}` (grid) and `w500`/`w780` (detail). Requests originate from the visitor's browser, so the site's origin never hits TMDB, and posters are never in the repo. Opt-in "local cache" mode is a script that downloads `w500` posters into `public/img/` and flips an `imageBase` config value; nothing else changes.
- **Licensing caveat to log in PROJECT.md**: a catalog whose purpose is selling DVDs is arguably "primary purpose is to create revenue". TMDB's stated test is monetization *of TMDB content* (ads, paid features); an unlisted personal listing with no ads, no checkout, no affiliate links is the same shape as the "free public webpage" TMDB staff explicitly approved. Keep it that way (no ads, no payments, attribution present). If the skill is later distributed to other sellers commercially, revisit. Same non-commercial clause applies to OMDb (CC BY-NC) and IMDb datasets.

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Next.js (App Router, `output: "export"`) | 16.3.5 (`latest` on npm 2026-09-21) | Static site generator + client routing | Matches the sibling repo's proven config (`output: "export"`, `trailingSlash: true`, `images.unoptimized: true`) and deploy workflow; `generateStaticParams` prerenders one `movies/<slug>/index.html` per title, so deep links work on GitHub Pages with no SPA-fallback hacks. Sibling is on `^15.2`; start this greenfield repo on 16.x (same App Router idioms, Turbopack default, min Node 20.9). Confidence: HIGH |
| React / React DOM | 19.3.0 | UI | Next 16 peer range `^19.0.0`; sibling uses 19. Confidence: HIGH |
| TypeScript | ^5.9.3 (**not** 7.x) | Types | Next docs state min 5.1; TS 7.0.2 is the new Go-native compiler and Next's TS plugin/tooling compatibility is unverified; pin the 5.x line to match the sibling (`^5`). Confidence: MEDIUM |
| Tailwind CSS + `@tailwindcss/postcss` | 4.3.3 | Styling | Sibling uses Tailwind v4 via PostCSS plugin; CSS-first config, no `tailwind.config.js`. Confidence: HIGH |
| lucide-react | 1.47.0 | Icons | Sibling dependency; peer `react ^19`. Confidence: HIGH |
| zod | 4.6.5 | Validate `catalog.json` at build time and in the skill | Single schema is the contract between the ingestion skill and the site; `next build` fails loudly on a malformed entry instead of shipping a broken grid. Confidence: HIGH |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Plain `Array.filter/sort` + `useMemo` | n/a | Genre/rating/popularity filter and title search | Default. ~500 items x a few string fields is microseconds; no index, no bundle cost. Normalize titles (lowercase, strip diacritics/articles) once at build. |
| Fuse.js | 7.5.0 (Apache-2.0, 20.5k stars, pushed 2026-08) | Typo-tolerant title search | Only if UAT shows buyers misspell titles. MiniSearch 7.2.0 (~6 kB gz) is the alternative if bundle size matters more than fuzziness; not needed at this scale. |
| `p-throttle` | 8.1.1 | Rate-limit TMDB/OMDb calls in the skill's scripts | Always in ingestion. |
| `@anthropic-ai/sdk` | 0.127.0 | **Not needed** for the skill | The skill runs *inside* Claude Code; the `Read` tool renders JPG/PNG natively so the model reads spines directly. Only add the SDK if a headless CLI ingest (no Claude Code) is later required. |
| `moviedb-promise` | 4.0.8 (237 stars, pushed 2026-01) | Typed TMDB client | Optional. For 3 endpoints, a 60-line `fetch` wrapper with zod response schemas is smaller and has no dependency risk; recommend plain `fetch` (Node 22 global). |
| `sharp` | 0.35.4 | Resize posters in the opt-in local-cache script | Only in `scripts/cache-images.ts`; not a site dependency. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| Node.js | 22 LTS (local 22.22.1) | Next 16 requires >=20.9; set `actions/setup-node` to `'22'` (the Next.js GitHub Pages template uses 22). Sibling's `'20'` still works but is EOL April 2026. |
| GitHub Actions: `actions/checkout@v4`, `actions/setup-node@v4`, `actions/upload-pages-artifact@v3`, `actions/deploy-pages@v4` | Deploy `out/` to Pages | Copy sibling's `deploy.yml` verbatim, add `env: NEXT_PUBLIC_BASE_PATH` on the build step. |
| `tsx` | Run `scripts/*.ts` (ingest helpers, cache-images, validate) | Dev dependency; the skill invokes `npx tsx ${CLAUDE_SKILL_DIR}/scripts/...`. |
| ESLint (`eslint-config-next` 16.3.5) | Lint | Optional; Next 16 no longer lints in `next build`. Sibling has no linter; keep parity unless wanted. |
| Claude Code skill | Ingestion UX | `.claude/skills/ingest-dvds/SKILL.md` + `scripts/`; see Ingestion section. |

## Architecture-Relevant Configuration

### Base path for `/sale/dvds`

- `basePath` is inlined at build time (cannot change post-build) and auto-prefixes `next/link` and the router. It does **not** prefix `<img src>` or `public/` asset paths: expose it as `NEXT_PUBLIC_BASE_PATH` and prepend it manually for local assets (favicon, TMDB logo, cached posters). TMDB CDN URLs are absolute and unaffected.
- `trailingSlash: true` emits `out/movies/<slug>/index.html`; GitHub Pages resolves `/sale/dvds/movies/<slug>/` directly, so no `404.html` redirect trick is needed. `dynamicParams` is unsupported in export mode; every slug must come from `generateStaticParams()` reading `catalog.json`.
- Composition path (a) from PROJECT.md: the main site's workflow checks out this repo's release artifact into `out/sale/dvds/` before `upload-pages-artifact`. Path (b) subdomain: build with `NEXT_PUBLIC_BASE_PATH=""` and a `CNAME`. Default GitHub project Pages: `NEXT_PUBLIC_BASE_PATH="/dvd-seller"`. All three are the same build with a different env var; this matches the official `nextjs/deploy-github-pages` template (`basePath: process.env.PAGES_BASE_PATH`).

### Data layer

- **Location:** `data/catalog.json` at repo root (not under `public/`, so it is bundled at build, not fetched at runtime). Import it in Server Components (`import catalog from "@/data/catalog.json"`) and validate with `CatalogSchema.parse()` in `lib/catalog.ts`; both the grid page and `generateStaticParams` consume the parsed array.
- **Build-time import, not runtime fetch.** At a few hundred entries (~1-2 kB each => <1 MB) the whole catalog is inlined into the page payload once and filtered in a Client Component. Runtime `fetch("/catalog.json")` would add a request, a loading state and a basePath foot-gun for no benefit. Revisit only past ~5,000 titles.
- **Schema (zod, shared by site and skill):** `id` (slug), `tmdbId`, `imdbId`, `title`, `year`, `runtime`, `overview`, `genres[]`, `popularity`, `tmdbRating`, `imdbRating?`, `director`, `cast[] {name, character, profilePath}` (top 8), `posterPath`, `backdropPath`, `sale {status: "available"|"sold"|"pending", price?, condition?, format: "DVD"|"Blu-ray"|"4K", edition?, notes?}`, `source {photo, confidence, ingestedAt}`. Keep TMDB path fragments, not full URLs.
- Add `scripts/validate.ts` (`npm run validate`) that the skill runs after every append and that CI runs before `next build`.

### Ingestion skill

- **Layout:** `.claude/skills/ingest-dvds/SKILL.md` (frontmatter: `name`, `description` with trigger phrases like "ingest DVD photos", `argument-hint: <photo paths>`, `disable-model-invocation: true` because it writes files, `allowed-tools: Read Bash(npx tsx ${CLAUDE_SKILL_DIR}/scripts/*)`), plus `scripts/search.ts`, `scripts/enrich.ts`, `scripts/append.ts`, and `reference/spine-reading-guide.md` (rotated text, box sets, "Special Edition" noise words). Keep SKILL.md under 500 lines; detail goes in reference files loaded on demand. Ship as a plugin later by adding `.claude-plugin/plugin.json` so other sellers install it with one command.
- **Vision:** the model reads each photo with Claude Code's built-in `Read` tool (renders images) and emits a candidate list `{rawText, title, yearHint, confidence}`. No OCR library, no Anthropic SDK call: this is why a Claude Code skill is the right packaging versus a standalone CLI.
- **Lookup:** `scripts/search.ts` takes the candidate list, throttles TMDB `search/movie` at 4 req/s, and returns top-3 matches with scores. The skill presents any candidate where confidence < 0.8 or the top two TMDB results are within 10% popularity of each other (remakes, editions) as a confirm-list to the seller, then calls `enrich.ts` (details + credits + external_ids, optional OMDb rating) and `append.ts` (zod-validate, dedupe on `tmdbId`, sort, write, run `validate`).
- **Config:** `TMDB_API_TOKEN`, optional `OMDB_API_KEY` from env / `.env.local`; catalog path from `dvd-seller.config.json` with a sane default, so nothing is Patrick-specific.

## Installation

# Core (site)

# Ingestion scripts (used by the skill; also site dev deps)

# Optional

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Next.js 16 static export | Astro 5 | If this were a fresh project with no sibling to match, Astro's content collections + islands would be the leaner fit for a JSON-driven catalog. Rejected here because the hard constraint is convention parity and composable deploy with patrickclery.github.io. |
| Next.js 16 static export | Vite + React Router (pure SPA) | Simpler mental model, but GitHub Pages has no rewrite rules, so per-movie deep links need the `404.html` copy hack and lose per-page HTML for link previews. Next prerenders every detail page. |
| Next.js 16 | Next.js 15.x (sibling's exact major) | Only if the two repos must share a `package.json` lockfile or a monorepo. 15.x is now the `backport` tag; both share identical `next.config` semantics for export. |
| Plain array filter | Fuse.js / MiniSearch | Fuse if fuzzy matching is a validated need; MiniSearch if the catalog grows to thousands and prefix search latency shows. |
| TMDB + optional OMDb | OMDb only | Never: no headshots, no popularity, poster API paywalled, 1k/day. |
| TMDB + optional OMDb | IMDb non-commercial datasets for ratings | Viable offline: `title.ratings.tsv.gz` is a daily bulk download and avoids OMDb's quota. Adds a ~7 MB download step to the skill; adopt only if OMDb's 1,000/day becomes a bottleneck. |
| Build-time JSON import | Runtime `fetch` of `/catalog.json` | Only if editing the catalog without rebuilding becomes a requirement (it does not; the skill commits, CI rebuilds). |
| Claude Code skill (in-agent vision) | Standalone CLI with `@anthropic-ai/sdk` vision calls | If sellers who do not use Claude Code need ingestion. Costs API spend per photo and duplicates what the agent already does. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Scraping imdb.com HTML or Amazon-hosted IMDb poster URLs | Violates IMDb terms; anti-bot blocking and rate limiting; images not licensed for reuse | TMDB API + `image.tmdb.org` CDN; link out to `https://www.imdb.com/title/<imdb_id>/` |
| IMDb official API (AWS Data Exchange) | Enterprise, contact-sales pricing; no free tier in 2026 | TMDB for metadata, OMDb or IMDb non-commercial TSVs for the rating number |
| `next/image` with default loader, ISR, Server Actions, route handlers reading `Request`, `dynamicParams: true`, middleware/proxy, `rewrites`/`redirects` | All unsupported under `output: "export"`; build errors or silent no-ops on Pages | `images.unoptimized: true` + `<img loading="lazy">`; `generateStaticParams`; static JSON |
| Committing hundreds of posters to the repo by default | Repo bloat, 1 GB Pages limit, 100 GB/mo soft bandwidth, and TMDB's 6-month cache clause | CDN fragments in JSON; opt-in `scripts/cache-images.ts` |
| Any server/DB app from the prior-art list (DVinyl, Movary, Yamtrack, Ryot) | Require Docker + database = paid hosting | Build the ~2-page static UI |
| TypeScript 7.x today | Native compiler is new; Next's TS plugin and `next build` type-check path unverified against it | `typescript@^5.9` |
| Node 20 in CI going forward | EOL April 2026; Next 16 min is 20.9 so it works, but pick a supported line | `actions/setup-node` with `'22'` |
| Firebase/Supabase for sale state, comments, or contact forms | Introduces a hosted backend, violating zero-hosting | `sale.status` in JSON; "contact seller" is a `mailto:`/link |
| Displaying TMDB `vote_average` labelled as "IMDb rating" | Misrepresents source and breaches attribution intent | Label ratings by source; fetch `imdbRating` from OMDb when available |

## Stack Patterns by Variant

- Build with `NEXT_PUBLIC_BASE_PATH=/sale/dvds`; main-site workflow downloads this repo's `out/` artifact into `out/sale/dvds/`.
- Because both are static exports, there is no runtime coupling; only the main site's workflow changes.
- `NEXT_PUBLIC_BASE_PATH=/dvd-seller` or `""` + `public/CNAME`.
- Same code; only the env var and optional CNAME differ.
- `npm run cache-images` downloads `w500` posters and `w185` headshots into `public/img/tmdb/`, writes `data/image-manifest.json`; `lib/images.ts` reads `NEXT_PUBLIC_IMAGE_BASE` (`https://image.tmdb.org/t/p` vs `${basePath}/img/tmdb`).
- Keep attribution; TMDB permits caching data/images up to 6 months, so the script records `cachedAt` and warns when stale.
- Skill skips enrichment; UI shows TMDB rating with a "TMDB" badge and hides IMDb-rating sort or sorts by TMDB rating with clear labelling.

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| next@16.3.5 | react@19.3.0, react-dom@19.3.0 | Peer `^19.0.0` verified via `npm view next@latest peerDependencies` |
| next@16.3.5 | node >=20.9 | Official installation docs; use 22 LTS |
| next@16.3.5 | typescript >=5.1 | Official docs; 7.x unverified, avoid |
| tailwindcss@4.3.3 | @tailwindcss/postcss@4.3.3 | Same version line; CSS-first `@import "tailwindcss"`; no `tailwind.config.js` |
| lucide-react@1.47.0 | react ^16.5 - ^19 | Peer range verified |
| zod@4.6.5 | TS 5.x | zod 4 API (`z.object`, `.parse`) unchanged for this use; import from `"zod"` |
| next@16 `output: "export"` | `trailingSlash: true` + `generateStaticParams` | Required combo for GitHub Pages deep links |

## Sources

- npm registry (`npm view <pkg> version` / `dist-tags` / `peerDependencies`, 2026-09-21) — versions for next, react, tailwindcss, typescript, zod, fuse.js, minisearch, lucide-react, p-throttle, sharp, moviedb-promise, @anthropic-ai/sdk. Confidence: HIGH (registry is authoritative)
- https://nextjs.org/docs/app/guides/static-exports (v16.3.5, updated 2026-08-25) — supported/unsupported features, `trailingSlash`, GitHub Pages template link. Confidence: MEDIUM (official docs via webfetch)
- https://nextjs.org/docs/app/api-reference/config/next-config-js/basePath — build-time inlining, Link vs Image behaviour. Confidence: MEDIUM
- https://nextjs.org/docs/app/getting-started/installation — Node >=20.9, TS >=5.1, Turbopack default, `next build` no longer lints. Confidence: MEDIUM
- https://github.com/nextjs/deploy-github-pages (157 stars, pushed 2025-12) — `basePath: process.env.PAGES_BASE_PATH`, Node 22 workflow. Confidence: MEDIUM
- https://developer.themoviedb.org/docs/rate-limiting — ~40 req/s, 429 handling. Confidence: MEDIUM
- https://www.themoviedb.org/api-terms-of-use and https://developer.themoviedb.org/docs/faq — attribution text/logo, non-commercial definition, 6-month cache, image-hosting prohibition. Confidence: MEDIUM
- https://www.themoviedb.org/talk/6a37cad7bb2a89e12c80dc6e and https://www.themoviedb.org/talk/697df2a0576e95a402e4e71e — TMDB staff (Travis Bell) on free public non-monetized sites and the $149/mo commercial tier. Confidence: MEDIUM (staff replies on official forum)
- https://developer.themoviedb.org/docs/image-basics, /docs/append-to-response, /docs/authentication-application — image URL construction, `append_to_response=credits,external_ids`, Bearer token on v3. Confidence: MEDIUM
- https://www.omdbapi.com/ and https://www.omdbapi.com/apikey.aspx — "FREE! (1,000 daily limit)", Poster API patrons-only, CC BY-NC 4.0. Confidence: MEDIUM
- https://data.imdb.com/ and https://data.imdb.com/non-commercial-datasets/ — GraphQL API via AWS Data Exchange only, dataset list, daily refresh, non-commercial terms, no images. Confidence: MEDIUM
- https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits — 1 GB site, 100 GB/mo soft bandwidth, 10 builds/hr soft. Confidence: MEDIUM
- https://code.claude.com/docs/en/skills — SKILL.md frontmatter, `${CLAUDE_SKILL_DIR}`, plugin packaging, 500-line guidance. Confidence: MEDIUM
- GitHub API via `gh api repos/...` and `gh search repos` (2026-09-21) — all prior-art stars, push dates, licenses, archived flags. Confidence: HIGH
- Sibling repo `~/github.com/patrickclery/patrickclery.github.io` (`package.json`, `next.config.ts`, `.github/workflows/deploy.yml`) — conventions to match. Confidence: HIGH (read directly)
- npmtrends / devpick / npm-compare comparisons — MiniSearch vs Fuse.js sizing. Confidence: LOW (third-party blogs; used only for a non-critical optional choice)

<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->

## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->

## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->

## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:

- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->

## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
