# Architecture Research

**Domain:** Static-data catalog SPA (DVD collection for sale) with an offline, agent-driven ingestion pipeline
**Researched:** 2026-09-21
**Confidence:** MEDIUM — core claims come from official Next.js / GitHub / TMDB / Claude Code docs fetched directly and cross-checked against the official `nextjs/deploy-github-pages` template and live `curl` probes of the TMDB CDN. The confidence seam rates `webfetch` as LOW; findings are tagged per seam below but every load-bearing claim has two independent official sources.

## Standard Architecture

The system is two loosely-coupled halves that share exactly one contract: `data/catalog.json` and its schema.

- **Ingestion half (offline, seller's machine, needs API keys):** photos → vision → TMDB/OMDb → validated JSON → git commit.
- **Publishing half (CI + visitor browser, no keys, no API calls):** JSON → Next.js static export → GitHub Pages → browser filters in memory and hotlinks images from `image.tmdb.org`.

Nothing in the publishing half ever talks to TMDB, OMDb, or IMDb. The only runtime network traffic a visitor generates is to GitHub Pages (HTML/JS/CSS) and to TMDB's image CDN (posters, headshots).

### System Overview

```
┌──────────────────────────────── SELLER'S MACHINE (offline ingest) ─────────────────────────────┐
│                                                                                                 │
│  spine photos ──► Claude Code skill  ──► scripts/ingest.ts ──► scripts/enrich.ts ──► validate  │
│  (*.jpg)          .claude/skills/        (TMDB search +        (details+credits+     (zod /     │
│                   dvd-ingest/SKILL.md     disambiguation)        external_ids, OMDb)  schema)   │
│                        │                        │                     │                 │       │
│                        │ vision reads titles    │ .cache/tmdb/*.json  │ .cache/omdb/    │       │
│                        │ + confidence           │ (on-disk response   │ p-limit throttle│       │
│                        ▼                        ▼  cache)             ▼                 ▼       │
│                   candidate list ──────────────────────────────► dedupe ──► append ──► git commit│
│                   (seller confirms                                          data/catalog.json    │
│                    low-confidence)                                                               │
└────────────────────────────────────────────────┬────────────────────────────────────────────────┘
                                                 │ git push (master)
┌────────────────────────────────────────────────▼──────────────── GITHUB ACTIONS (build) ───────┐
│  checkout ─► npm ci ─► NEXT_PUBLIC_BASE_PATH=<path> next build ─► out/ ─► upload-pages-artifact  │
│                        │                                                                        │
│                        ├─ generateStaticParams() reads data/catalog.json → out/movie/<slug>/    │
│                        ├─ index page embeds card projection of catalog → out/index.html         │
│                        └─ optional: IMAGE_MODE=local uses public/posters/** instead of CDN      │
└────────────────────────────────────────────────┬────────────────────────────────────────────────┘
                                                 │ deploy-pages
┌────────────────────────────────────────────────▼──────────────── VISITOR BROWSER ──────────────┐
│  GitHub Pages (static HTML/JS/CSS) ──► React hydrates ──► client-side filter/sort/search        │
│         │                                     │              (state mirrored to ?genre=&sort=)  │
│         │  /sale/dvds/movie/603-the-matrix/   │                                                 │
│         ▼                                     ▼                                                 │
│  detail page (prebuilt HTML)          <img loading="lazy" src="https://image.tmdb.org/t/p/w342/…"> │
│                                       (TMDB CDN, cache-control max-age=1y; no origin calls)     │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| `data/catalog.json` | Single source of truth. One entry per physical DVD title. Metadata + sale state. Committed; git history is the audit log. | Plain JSON array wrapped in `{ "$schema", "version", "updatedAt", "movies": [...] }` |
| `data/catalog.schema.json` + `src/lib/schema.ts` | The contract both halves obey. Zod schema is canonical; JSON Schema is emitted from it for editor validation and for the skill to read. | `zod` → `zod-to-json-schema` (or hand-maintained JSON Schema if the stack researcher prefers Ajv) |
| `.claude/skills/dvd-ingest/` | Seller-facing agent workflow: accepts photo paths, does the vision read itself (Claude is multimodal), orchestrates the scripts, asks the seller to confirm ambiguous titles, commits. No API keys inside — they come from env. | `SKILL.md` + `reference.md` (disambiguation rules) + calls `${CLAUDE_PROJECT_DIR}/scripts/*.ts` |
| `scripts/ingest.ts` | Given candidate titles (+ optional year hints), search TMDB, return ranked matches with confidence; never writes the catalog. | `tsx` script, TMDB `/search/movie`, on-disk cache |
| `scripts/enrich.ts` | Given a confirmed `tmdbId`, fetch details + credits + external_ids in one call (`append_to_response`), OMDb rating by `imdb_id`, map to the catalog entry shape. | TMDB `/movie/{id}?append_to_response=credits,external_ids`, OMDb `?i=tt…` |
| `scripts/validate.ts` | Parse `catalog.json` against the schema, check slug uniqueness, `tmdbId+edition` uniqueness, poster path format. Fails CI if broken. | Zod `safeParse`, exit 1 on error; wired into `npm run build` as `prebuild` |
| `scripts/cache-images.ts` | Opt-in: download every referenced poster/headshot into `public/posters/`, `public/profiles/`. Idempotent; skips existing files. | `undici` fetch + `p-limit(4)`; writes nothing into `catalog.json` (paths stay TMDB-relative) |
| `scripts/mark-sold.ts` | Flip `sale.status`, set `soldAt`, optional price; the "seller marks item sold" path. | Small CLI; also callable by the skill (`/dvd-ingest sold "The Matrix"`) |
| `src/lib/catalog.ts` | Server-side-only accessor: imports the JSON, exposes `getAllMovies()`, `getMovie(slug)`, `getCards()` (slim projection for the grid), `getGenres()`. | Static `import catalog from "@/data/catalog.json"`; runs at build time only |
| `src/lib/images.ts` | Builds image URLs: `posterUrl(path, size)` returns `https://image.tmdb.org/t/p/w342${path}` or `${basePath}/posters/w342${path}` depending on `NEXT_PUBLIC_IMAGE_MODE`. | Pure function; the *only* place that knows about the CDN |
| `src/lib/base-path.ts` | `withBasePath("/posters/x.jpg")` for non-`next/link` URLs (plain `<img>`, `<meta og:image>`, manifest). | Reads `process.env.NEXT_PUBLIC_BASE_PATH ?? ""` |
| `src/app/page.tsx` | Server component: loads cards, renders `<Suspense><CatalogBrowser cards={…} /></Suspense>`. | Static HTML shell + RSC payload |
| `src/components/CatalogBrowser.tsx` | Client component: filter/sort/search in memory; mirrors state to URL search params; renders `MovieCard` grid. | `useSearchParams` + `useRouter().replace` |
| `src/app/movie/[slug]/page.tsx` | `generateStaticParams()` from catalog; `dynamicParams = false`; Plex-style detail. | One `index.html` per movie |
| `src/app/not-found.tsx` | Emitted as `out/404.html`; GitHub Pages serves it for unknown paths. | Links back to the catalog root via `<Link href="/">` (basePath-aware) |
| `.github/workflows/deploy.yml` | Mirror of the sibling repo: Node 20, `npm ci`, `npm run build`, `upload-pages-artifact` from `out/`, `deploy-pages@v4`. Sets `NEXT_PUBLIC_BASE_PATH`. | Copy from `patrickclery.github.io`, add env + `prebuild` validate |

## Recommended Project Structure

```
dvd-seller/
├── .claude/
│   └── skills/
│       └── dvd-ingest/
│           ├── SKILL.md                 # workflow: read spines → confirm → run scripts → commit
│           ├── reference.md             # disambiguation rules, edition handling, confidence rubric
│           └── examples.md              # sample sessions (one photo, many photos, "sold")
├── .github/workflows/deploy.yml         # copy of sibling; adds NEXT_PUBLIC_BASE_PATH + validate
├── .cache/                              # gitignored: tmdb/, omdb/ raw API responses (JSON per request)
├── data/
│   ├── catalog.json                     # THE data file (committed)
│   └── catalog.schema.json              # generated from src/lib/schema.ts (committed for editors/skill)
├── public/
│   ├── .nojekyll                        # required: _next/ is underscore-prefixed
│   ├── posters/                         # only populated when IMAGE_MODE=local (gitignored by default)
│   └── profiles/
├── scripts/
│   ├── ingest.ts                        # titles → TMDB candidates (JSON out, no writes)
│   ├── enrich.ts                        # tmdbId → full entry (TMDB + OMDb), appends to catalog
│   ├── validate.ts                      # schema + uniqueness checks; prebuild hook
│   ├── cache-images.ts                  # opt-in local image mirror
│   ├── mark-sold.ts                     # sale-state edits
│   └── lib/
│       ├── tmdb.ts                      # typed client, p-limit throttle, disk cache
│       ├── omdb.ts                      # typed client, daily-budget guard
│       └── cache.ts                     # keyed fs cache (sha of URL) — shared by both clients
├── src/
│   ├── app/
│   │   ├── layout.tsx                   # fonts, TMDB attribution in footer
│   │   ├── page.tsx                     # catalog grid (server shell + client browser)
│   │   ├── not-found.tsx                # → out/404.html
│   │   └── movie/[slug]/page.tsx        # generateStaticParams over catalog.json
│   ├── components/
│   │   ├── CatalogBrowser.tsx           # 'use client' — filters, sort, search, URL state
│   │   ├── FilterBar.tsx
│   │   ├── MovieCard.tsx
│   │   ├── MovieDetail.tsx
│   │   ├── CastList.tsx
│   │   └── Poster.tsx                   # <img loading="lazy"> wrapper using lib/images
│   └── lib/
│       ├── schema.ts                    # Zod: Movie, Catalog — the shared contract
│       ├── catalog.ts                   # build-time accessors + card projection
│       ├── images.ts                    # CDN vs local URL builder
│       ├── base-path.ts                 # withBasePath()
│       └── filters.ts                   # pure filter/sort functions (unit-testable)
├── next.config.ts                       # output export, trailingSlash, basePath from env
└── package.json                         # "prebuild": "tsx scripts/validate.ts"
```

### Structure Rationale

- **`data/` at repo root, not `src/`:** The skill and scripts edit it; the SPA only reads it. Keeping it out of `src/` makes the "one-way street" obvious and keeps Next's file watcher from treating skill edits as app changes.
- **`scripts/lib/` separate from `src/lib/`:** Scripts run under Node with API keys and filesystem access; `src/lib/` is bundled into the browser. Sharing only `src/lib/schema.ts` (pure Zod, no I/O) enforces that no key or fetch logic can leak into the client bundle.
- **`.claude/skills/dvd-ingest/` inside the repo:** Project-scoped skills are discovered from the repo root and committed, so any seller who clones the repo gets `/dvd-ingest` (verified against Claude Code skills docs). The skill references scripts via `${CLAUDE_PROJECT_DIR}/scripts/…`, so no absolute Patrick-specific paths.
- **`.cache/` gitignored:** Raw TMDB/OMDb responses are cached on disk so re-running enrichment (e.g. after a schema change) costs zero API calls. Not committed — it is derived data.

## Architectural Patterns

### Pattern 1: Build-time data import, client-side projection

**What:** The catalog JSON is imported in server components at build time. The index page passes only a slim "card" projection (≈10 fields) to the client component; detail pages get the full entry but are prebuilt per movie.
**When to use:** Any static catalog under a few thousand entries. At 400 entries, cards ≈ 300 bytes each → ~120 KB embedded in `index.html`/RSC payload; full entries with cast would be 3–5× that, which is why the projection matters.
**Trade-offs:** No runtime fetch, works offline after first load, trivially cacheable. Cost: any catalog change requires a rebuild (acceptable — that is the deploy model anyway).

```typescript
// src/lib/catalog.ts  (server/build only — never imported from a 'use client' file)
import raw from "../../data/catalog.json";
import { CatalogSchema, type Movie } from "./schema";

const catalog = CatalogSchema.parse(raw); // throws at build if the skill wrote bad data

export function getAllMovies(): Movie[] { return catalog.movies; }
export function getMovie(slug: string) { return catalog.movies.find(m => m.slug === slug); }
export function getCards() {
  return catalog.movies.map(({ slug, title, year, posterPath, genres, ratings, popularity, sale }) =>
    ({ slug, title, year, posterPath, genres, imdb: ratings.imdb, popularity, status: sale.status }));
}
```

### Pattern 2: Static deep links via `generateStaticParams` + `dynamicParams = false`

**What:** `/movie/[slug]/page.tsx` enumerates every slug from the catalog at build time. With `trailingSlash: true` the export emits `out/movie/<slug>/index.html`, which GitHub Pages serves directly for `/movie/<slug>/` with no rewrite rules. Unknown slugs fall through to `out/404.html`, which GitHub Pages uses as the custom 404 page.
**When to use:** Always, for static export — dynamic routes without `generateStaticParams` are unsupported in `output: "export"` (official docs).
**Trade-offs:** Slugs must be stable; renaming a title would break shared links. Solve by storing `slug` in the JSON at ingest time and never regenerating it.

```typescript
// src/app/movie/[slug]/page.tsx
import { getAllMovies, getMovie } from "@/lib/catalog";
import { notFound } from "next/navigation";

export const dynamicParams = false;

export function generateStaticParams() {
  return getAllMovies().map(m => ({ slug: m.slug }));
}

export default async function MoviePage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const movie = getMovie(slug);
  if (!movie) notFound();
  return <MovieDetail movie={movie} />;
}
```

**Slug design:** `${tmdbId}-${kebab(title)}` → `603-the-matrix`. The numeric prefix guarantees uniqueness across remakes (`1091-the-thing` vs `60935-the-thing`) and makes the slug self-describing; the kebab title is for humans and SEO. Store it; do not derive it at render time.

### Pattern 3: URL-mirrored filter state behind a Suspense boundary

**What:** `CatalogBrowser` (client component) reads `useSearchParams()` as the source of truth for `genre`, `sort`, `q`, `minRating`, `status`, and writes back with `router.replace(pathname + "?" + params, { scroll: false })`. The server page wraps it in `<Suspense fallback={<GridSkeleton/>}>`.
**When to use:** Any static page that needs shareable filtered views. The Suspense wrapper is **mandatory**: a production `next build` fails with `missing-suspense-with-csr-bailout` when `useSearchParams` is used on a prerendered route without one; `next dev` hides the error (official docs).
**Trade-offs:** The grid is client-rendered after hydration (the fallback is what is in the static HTML). Fine for a browsing UI; the per-movie detail pages remain fully prerendered for link previews.

```typescript
// src/components/CatalogBrowser.tsx
'use client';
import { useRouter, usePathname, useSearchParams } from "next/navigation";
import { applyFilters, parseFilters } from "@/lib/filters";

export function CatalogBrowser({ cards }: { cards: Card[] }) {
  const router = useRouter(), pathname = usePathname(), sp = useSearchParams();
  const filters = parseFilters(sp);                 // pure: URLSearchParams → Filters
  const visible = applyFilters(cards, filters);     // pure: unit-tested, no React
  const set = (patch: Partial<Filters>) => {
    const next = new URLSearchParams(sp.toString());
    for (const [k, v] of Object.entries(patch)) v ? next.set(k, String(v)) : next.delete(k);
    router.replace(`${pathname}?${next}`, { scroll: false });
  };
  return <><FilterBar filters={filters} onChange={set} /><Grid cards={visible} /></>;
}
```

### Pattern 4: Environment-driven `basePath` with a single URL helper

**What:** `next.config.ts` sets `basePath: process.env.NEXT_PUBLIC_BASE_PATH || undefined`. `next/link` and the router prefix automatically; everything else (plain `<img src>` for local posters, `og:image`, favicon) goes through `withBasePath()`. `assetPrefix` is not needed — `basePath` already prefixes `_next/` assets.
**When to use:** Any static site that must be mountable at more than one path. This is exactly what the official `nextjs/deploy-github-pages` template does (`basePath: process.env.PAGES_BASE_PATH`).
**Trade-offs:** `basePath` is inlined at build time; changing the mount point means a rebuild, not a config flip. Never hard-code `/sale/dvds` in components.

```typescript
// next.config.ts
import type { NextConfig } from "next";
const basePath = process.env.NEXT_PUBLIC_BASE_PATH?.replace(/\/$/, "") || undefined;
const nextConfig: NextConfig = {
  output: "export",
  trailingSlash: true,
  basePath,
  images: { unoptimized: true },
};
export default nextConfig;

// src/lib/base-path.ts
export const BASE_PATH = process.env.NEXT_PUBLIC_BASE_PATH?.replace(/\/$/, "") ?? "";
export const withBasePath = (p: string) => `${BASE_PATH}${p.startsWith("/") ? p : `/${p}`}`;
```

Do **not** use `actions/configure-pages` with `static_site_generator: next` — it rewrites `next.config` to inject `basePath = "/<repo-name>"`, which fights the env-driven approach and would silently pin the build to `/dvd-seller`.

### Pattern 5: Image mode switch in one module

**What:** `src/lib/images.ts` is the only code that knows image URLs. `NEXT_PUBLIC_IMAGE_MODE=remote` (default) → `https://image.tmdb.org/t/p/w342${posterPath}`; `local` → `withBasePath("/posters/w342" + posterPath)`. `catalog.json` always stores the TMDB-relative `posterPath` (`/abc123.jpg`), never a full URL, so switching modes never touches data.
**When to use:** Default remote. Use local only if TMDB CDN access becomes a concern or the seller wants a fully self-contained archive.
**Trade-offs (measured 2026-09-21 via `curl -I`):** w342 posters are 63–91 KB (≈75 KB avg); TMDB serves them with `cache-control: public, max-age=31536000`, so hotlinking is CDN-friendly and cheap. Local mode for ~400 titles ≈ 30 MB of posters plus ≈ 30 MB of w185 headshots if cast is cached too — well under GitHub's 1 GB soft limit but every refresh churns git history. Recommend caching posters only, not headshots, in local mode.

```typescript
// src/lib/images.ts
import { withBasePath } from "./base-path";
const TMDB = "https://image.tmdb.org/t/p";
const MODE = process.env.NEXT_PUBLIC_IMAGE_MODE === "local" ? "local" : "remote";
export type PosterSize = "w185" | "w342" | "w500";
export type ProfileSize = "w45" | "w185";

export function posterUrl(path: string | null, size: PosterSize = "w342") {
  if (!path) return withBasePath("/placeholder-poster.svg");
  return MODE === "local" ? withBasePath(`/posters/${size}${path}`) : `${TMDB}/${size}${path}`;
}
export function profileUrl(path: string | null, size: ProfileSize = "w185") {
  if (!path) return withBasePath("/placeholder-headshot.svg");
  return MODE === "local" ? withBasePath(`/profiles/${size}${path}`) : `${TMDB}/${size}${path}`;
}
```

Use a plain `<img loading="lazy" decoding="async" width={342} height={513}>` rather than `next/image`: with `images.unoptimized` there is no benefit, and `next/image` needs `remotePatterns` config plus manual basePath handling.

### Pattern 6: Throttled, cached, key-isolated API clients (ingestion only)

**What:** `scripts/lib/tmdb.ts` and `omdb.ts` wrap `fetch` with (a) a `p-limit(4)` concurrency gate (TMDB's current soft ceiling is ~40 req/s with 429s beyond it; 4 concurrent is far below), (b) a disk cache keyed by SHA-256 of the URL under `.cache/`, and (c) an OMDb daily budget counter (free key = 1,000 req/day). Keys come from `TMDB_API_KEY` / `OMDB_API_KEY` env (or `.env.local`, gitignored).
**When to use:** All ingestion. The visitor site imports none of this.
**Trade-offs:** Slightly more code than raw `fetch`; pays for itself the first time you re-enrich the whole catalog after adding a field (zero API calls on a warm cache).

```typescript
// scripts/lib/tmdb.ts
import pLimit from "p-limit";
import { cached } from "./cache";
const limit = pLimit(4);
const KEY = process.env.TMDB_API_KEY ?? (() => { throw new Error("TMDB_API_KEY missing"); })();

export const tmdb = <T>(path: string, params: Record<string, string> = {}) =>
  limit(() => cached<T>(`tmdb:${path}:${JSON.stringify(params)}`, async () => {
    const url = new URL(`https://api.themoviedb.org/3${path}`);
    Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));
    const res = await fetch(url, { headers: { Authorization: `Bearer ${KEY}` } });
    if (res.status === 429) { await new Promise(r => setTimeout(r, 2000)); return tmdb<T>(path, params); }
    if (!res.ok) throw new Error(`TMDB ${res.status} ${path}`);
    return res.json() as Promise<T>;
  }));

// one call per movie for everything the catalog needs:
export const movieBundle = (id: number) =>
  tmdb<TmdbMovieBundle>(`/movie/${id}`, { append_to_response: "credits,external_ids" });
```

## Data Flow

### Ingestion Flow (seller, offline)

```
/dvd-ingest photos/shelf-01.jpg photos/shelf-02.jpg
    │
    ▼
[1] Vision read (Claude, inside the skill turn — no external vision API)
    per photo → [{ rawTitle, yearHint?, editionHint?, confidence: 0..1 }]
    │  rotated-90° spine text; box sets expand to N titles
    ▼
[2] scripts/ingest.ts --titles candidates.json  →  candidates-ranked.json
    TMDB /search/movie (title[, year]) → top 3 per title with { tmdbId, title, year, popularity }
    │  auto-accept when: exactly one result OR top result year matches hint & score gap large
    ▼
[3] Disambiguation (skill asks seller via AskUserQuestion)
    only for: confidence < 0.8, multiple plausible years (remakes), editions, unreadable spines
    → confirmed.json  [{ tmdbId, edition?, condition?, price? }]
    ▼
[4] scripts/enrich.ts --input confirmed.json
    for each tmdbId (p-limit 4, disk cache):
      TMDB /movie/{id}?append_to_response=credits,external_ids
      OMDb ?i={imdb_id}  → imdbRating, imdbVotes, Ratings[]
    map → Movie entry (schema below); slug = `${tmdbId}-${kebab(title)}`
    ▼
[5] Dedupe + validate
    reject if (tmdbId, edition) already present → report "already in catalog"
    CatalogSchema.parse(newCatalog) — hard fail, nothing written on error
    ▼
[6] Write data/catalog.json (sorted by tmdbId for stable diffs), bump updatedAt
    ▼
[7] git add data/catalog.json && git commit -m "catalog: add N titles from shelf-01/02"
    (skill shows summary; seller pushes when ready)
    ▼
[8] push → Actions build → Pages deploy (2–3 min)
```

### Sale-state Flow

```
seller: /dvd-ingest sold "The Matrix"   (or: npx tsx scripts/mark-sold.ts 603-the-matrix --price 5)
    → resolve by slug/title (fuzzy; asks if ambiguous)
    → sale.status = "sold", sale.soldAt = today
    → validate → write → commit → push → redeploy
Manual edit of data/catalog.json is equally valid; `prebuild` validation catches typos in CI.
```

### Request Flow (visitor)

```
GET https://patrickclery.com/sale/dvds/?genre=Horror&sort=rating
    ↓ GitHub Pages serves out/index.html (static shell + skeleton grid)
    ↓ JS hydrates; CatalogBrowser reads ?genre=Horror&sort=rating
    ↓ applyFilters(cards) in memory (400 entries → sub-millisecond)
    ↓ grid renders; each <img loading="lazy"> fetches https://image.tmdb.org/t/p/w342/…
click card
    ↓ <Link href="/movie/603-the-matrix/"> → client navigation (prefetched RSC payload)
    ↓ or hard load: GET /sale/dvds/movie/603-the-matrix/ → out/movie/603-the-matrix/index.html
    ↓ detail page: poster w500, cast headshots w185, "View on IMDb" → https://www.imdb.com/title/tt0133093/
```

### State Management

```
URL search params (source of truth for filters)
    ↓ useSearchParams()
CatalogBrowser ──► parseFilters() ──► applyFilters(cards) ──► Grid
    ▲                                                            │
    └──── router.replace(?…) ◄──── FilterBar onChange ◄──────────┘
```

No global store. The catalog is props; filter state is the URL; nothing else is stateful. Back/forward and share links work for free.

### Key Data Flows

1. **Catalog → site:** one-directional, build-time only. `data/catalog.json` → `src/lib/catalog.ts` → server components → static HTML/RSC. The browser never fetches the JSON separately.
2. **Skill → catalog:** the only writer. Scripts are pure functions over inputs; the skill sequences them and owns the human-in-the-loop step.
3. **Image bytes:** browser → TMDB CDN directly (remote mode) or → GitHub Pages (local mode). Our origin never proxies images.

## Proposed `catalog.json` Entry Schema

Both the SPA (`src/lib/schema.ts`) and the skill (`data/catalog.schema.json`, generated from it) agree on this. Fields are grouped so the "metadata" half is regenerable from TMDB and the "seller" half (`disc`, `sale`) is hand-owned and must survive re-enrichment.

```typescript
// src/lib/schema.ts
import { z } from "zod";

export const CastMember = z.object({
  tmdbId: z.number().int(),
  name: z.string(),
  character: z.string().nullable(),
  profilePath: z.string().regex(/^\/[\w-]+\.(jpg|png)$/).nullable(),   // TMDB-relative
  order: z.number().int(),
});

export const Movie = z.object({
  // identity
  id: z.string(),                          // stable unique key; == slug unless duplicate editions ("603-the-matrix", "603-the-matrix--steelbook")
  slug: z.string().regex(/^\d+-[a-z0-9-]+(--[a-z0-9-]+)?$/),
  tmdbId: z.number().int(),
  imdbId: z.string().regex(/^tt\d{7,8}$/).nullable(),

  // core metadata (from TMDB; regenerable)
  title: z.string(),
  originalTitle: z.string().nullable(),
  year: z.number().int().nullable(),
  releaseDate: z.string().nullable(),      // ISO date
  runtimeMinutes: z.number().int().nullable(),
  tagline: z.string().nullable(),
  overview: z.string(),
  genres: z.array(z.string()),             // names, e.g. ["Action", "Science Fiction"]
  posterPath: z.string().regex(/^\/[\w-]+\.(jpg|png)$/).nullable(),
  backdropPath: z.string().nullable(),
  director: z.array(z.string()),           // 0..n (from credits.crew job === "Director")
  cast: z.array(CastMember).max(12),       // top-billed only; keeps JSON small
  popularity: z.number(),                  // TMDB popularity at ingest time
  ratings: z.object({
    imdb: z.number().min(0).max(10).nullable(),        // OMDb imdbRating
    imdbVotes: z.number().int().nullable(),
    tmdb: z.number().min(0).max(10).nullable(),        // vote_average
    tmdbVotes: z.number().int().nullable(),
    rottenTomatoes: z.number().int().min(0).max(100).nullable(),  // from OMDb Ratings[]
    metacritic: z.number().int().min(0).max(100).nullable(),
  }),

  // physical item (seller-owned; never overwritten by enrich)
  disc: z.object({
    format: z.enum(["DVD", "Blu-ray", "4K UHD"]).default("DVD"),
    edition: z.string().nullable(),        // "Special Edition", "Steelbook", "Director's Cut"
    region: z.string().nullable(),         // "1", "A", "Free"
    condition: z.enum(["new", "like-new", "good", "fair", "poor"]).nullable(),
    notes: z.string().nullable(),          // "case cracked, disc mint"
    boxSet: z.string().nullable(),         // group id if part of a set sold together
  }),

  // sale state (seller-owned)
  sale: z.object({
    status: z.enum(["available", "reserved", "sold"]).default("available"),
    priceCents: z.number().int().nonnegative().nullable(),
    currency: z.string().length(3).default("CAD"),
    soldAt: z.string().nullable(),         // ISO date
  }),

  // provenance (skill-written)
  ingest: z.object({
    sourcePhoto: z.string().nullable(),    // relative path or filename of the spine photo
    ocrTitle: z.string().nullable(),       // what the vision read said
    confidence: z.number().min(0).max(1),
    confirmedBy: z.enum(["auto", "seller"]),
    addedAt: z.string(),                   // ISO datetime
    enrichedAt: z.string(),                // ISO datetime — re-run enrich when stale
  }),
});

export const Catalog = z.object({
  $schema: z.string().optional(),
  version: z.literal(1),
  updatedAt: z.string(),
  attribution: z.literal("This product uses the TMDB API but is not endorsed or certified by TMDB."),
  movies: z.array(Movie),
});
export type Movie = z.infer<typeof Movie>;
export type Catalog = z.infer<typeof Catalog>;
```

Validation invariants enforced by `scripts/validate.ts` beyond the shape: `slug` unique; `(tmdbId, disc.edition)` unique; `id === slug`; every `posterPath` referenced exists under `public/posters/w342/` when `IMAGE_MODE=local`. Sort `movies` by `tmdbId` before writing so diffs are minimal and merge conflicts between two ingest sessions are rare.

## Deploy Topologies for the `/sale/dvds` Mount

A verified fact reshapes this decision: **GitHub project sites inherit the account's custom domain.** Because `patrickclery.github.io` has `CNAME = patrickclery.com`, enabling Pages on `patrickclery/dvd-seller` publishes it at `https://patrickclery.com/dvd-seller/` with zero coordination (GitHub docs: "the GitHub Pages site for that repository will be available at `www.octocat.com/octo-project`"). The repo name cannot contain a slash, so `/sale/dvds` specifically needs composition.

| Topology | URL | How | Pros | Cons |
|----------|-----|-----|------|------|
| **(a) Own project Pages site** | `patrickclery.com/dvd-seller/` | This repo's `deploy.yml`, `NEXT_PUBLIC_BASE_PATH=/dvd-seller` | Zero changes to the main site; independent deploys; ships day one; exact mirror of sibling workflow | Path is `/dvd-seller`, not `/sale/dvds` |
| **(b) Main-site composes** | `patrickclery.com/sale/dvds/` | Main-site workflow adds a job: `actions/checkout` with `repository: patrickclery/dvd-seller` + `path: dvd-seller`, `npm ci && NEXT_PUBLIC_BASE_PATH=/sale/dvds npm run build`, `cp -r dvd-seller/out out/sale/dvds`. This repo's workflow ends with `repository_dispatch` to the main repo (fine-grained PAT, `contents: read` + `actions: write`) | Exact target path; still one static artifact; main site owns the URL namespace | Couples two repos' deploys; main site must rebuild for every "sold" flip; main repo needs the PAT secret and ~15 lines of YAML |
| **(c) Subdomain** | `dvds.patrickclery.com` | `public/CNAME` in this repo + DNS CNAME → `patrickclery.github.io`; `NEXT_PUBLIC_BASE_PATH` empty | Fully independent; cleanest URLs; no basePath at all | Needs DNS; not the requested `/sale/dvds`; separate cookie/origin (irrelevant here) |

Rejected variants of (b): pulling the `github-pages` artifact cross-repo with `actions/download-artifact` (needs `actions:read` token, the artifact is a tarball built with a specific basePath, and it expires after 90 days) and git submodules (painful to bump). Building from source inside the main-site job is simpler and keeps both repos public-only.

**Recommendation:** Ship **(a) for v1**, build **(b) as the final "mount" phase**. Rationale: (a) validates the whole pipeline (ingest → JSON → build → Pages) with the exact workflow the user already trusts and no cross-repo secrets, and it is live at `patrickclery.com/dvd-seller/` immediately — an unlisted URL just as good for early buyers. (b) is then a pure deployment change: the same build with a different `NEXT_PUBLIC_BASE_PATH`, plus one job in the main repo. Because `basePath` is env-driven and every URL goes through `next/link` or `withBasePath()`, switching costs nothing in application code. Keep (a) running as a preview target even after (b) exists, or disable Pages on this repo once (b) is stable to avoid two canonical URLs.

**Switching later:** (a)→(b): add the compose job + dispatch step; (a)→(c): add `public/CNAME`, DNS record, unset base path. No source changes in either case.

## Suggested Build Order

Dependencies flow from the contract outward. Each step is independently verifiable.

1. **Schema + sample data** — `src/lib/schema.ts`, `data/catalog.schema.json`, `data/catalog.json` with 5–10 hand-enriched entries (run `enrich.ts` once manually, or hand-write). `scripts/validate.ts`. *Unblocks everything else; nothing downstream can be built without a real entry to render.*
2. **Static SPA skeleton** — `next.config.ts` (export, trailingSlash, env basePath, `.nojekyll`), `lib/catalog.ts`, `lib/images.ts`, `lib/base-path.ts`, grid page with Suspense-wrapped `CatalogBrowser`, `movie/[slug]` with `generateStaticParams`, `not-found.tsx`. Verify `NEXT_PUBLIC_BASE_PATH=/sale/dvds npm run build && npx serve out` renders under the prefix with working deep links and 404.
3. **Deploy (topology a)** — create repo via `gh`, copy sibling `deploy.yml`, add `prebuild` validate and `NEXT_PUBLIC_BASE_PATH=/dvd-seller`. Live URL exists from here on; every later phase ships continuously.
4. **Ingestion scripts** — `scripts/lib/{cache,tmdb,omdb}.ts`, `ingest.ts`, `enrich.ts`, `mark-sold.ts`. Testable from the CLI with a hand-typed title list before any vision work.
5. **Ingestion skill** — `.claude/skills/dvd-ingest/SKILL.md` orchestrating steps 1–7 of the ingestion flow, with the confidence rubric and disambiguation prompts. Run on the real shelf photos; this is where the catalog fills up.
6. **Filter/sort/search polish + detail UX** — with hundreds of real entries, tune `filters.ts`, genre chips, rating/popularity sorts, sold badges, responsive layout, TMDB attribution footer.
7. **Optional local image cache** — `scripts/cache-images.ts`, `NEXT_PUBLIC_IMAGE_MODE=local` path in `images.ts`, size check.
8. **Mount at `/sale/dvds` (topology b)** — compose job in the main-site repo + `repository_dispatch` from this repo.

Steps 2 and 4 can proceed in parallel once step 1 exists; step 5 depends on 4; step 8 depends on 3 having proven the build.

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 100–500 titles (this project) | Single JSON, whole card list embedded in `index.html`, in-memory filtering. No pagination, no search index. |
| 500–3,000 titles | Move cards to a static `catalog.cards.json` emitted by a `force-static` route handler and fetch it client-side with SWR so `index.html` stays small; virtualize the grid. Build time (one HTML per movie) is still fine at ~1–2 s per 100 pages. |
| 3,000+ titles | Shard `catalog.json` by first letter or genre; consider Pagefind or MiniSearch prebuilt index for search; consider dropping per-movie prerender in favor of a single client-routed detail view (loses static deep-link HTML — do not do this below 3k). |

### Scaling Priorities

1. **First bottleneck:** `index.html` payload size as the card list grows (~300 B/card). Fix by slimming the projection or externalizing cards to a JSON fetched on hydrate.
2. **Second bottleneck:** OMDb's 1,000/day cap during a bulk enrich of a large collection. Fix is already in the design: disk cache + a daily-budget guard that pauses and resumes the next day; TMDB alone gives `vote_average` as a fallback rating.

## Anti-Patterns

### Anti-Pattern 1: Fetching TMDB/OMDb from the browser

**What people do:** Call `api.themoviedb.org` client-side for ratings or posters "to keep data fresh."
**Why it's wrong:** Exposes the API key in the bundle, makes every visitor a rate-limit liability, and reintroduces a runtime dependency into a site whose whole point is zero moving parts.
**Do this instead:** Bake everything into `catalog.json` at ingest; re-run `enrich.ts` (cache-warm, cheap) when you want refreshed ratings and redeploy.

### Anti-Pattern 2: Storing full image URLs (or base-path-prefixed paths) in the data file

**What people do:** Write `"poster": "https://image.tmdb.org/t/p/w500/abc.jpg"` or `"/sale/dvds/posters/abc.jpg"` into JSON.
**Why it's wrong:** Locks the data to one CDN size, one image mode, and one mount path. Changing any of them means rewriting hundreds of entries.
**Do this instead:** Store TMDB-relative `posterPath` only; construct URLs in `lib/images.ts` at render time.

### Anti-Pattern 3: `useSearchParams` without Suspense (works in dev, fails in `next build`)

**What people do:** Read filter params in a client component at the top of the page.
**Why it's wrong:** `next build` for a prerendered route aborts with the CSR-bailout error; `next dev` never shows it, so it surfaces only in CI.
**Do this instead:** Wrap the browser component in `<Suspense fallback={…}>` in the server page from day one; keep filter logic in pure functions so the component stays thin.

### Anti-Pattern 4: Letting the skill call the APIs directly via ad-hoc `curl`

**What people do:** Write the SKILL.md so Claude issues raw `curl https://api.themoviedb.org/...` calls and pastes JSON into the catalog.
**Why it's wrong:** No throttling, no cache, no schema validation, and the agent "helpfully" fixes malformed JSON by hand. Also not reusable by other sellers.
**Do this instead:** The skill only sequences typed scripts and handles the human confirmation step. Scripts own network, cache, validation, and writes.

### Anti-Pattern 5: Regenerating slugs or overwriting seller-owned fields on re-enrich

**What people do:** Rebuild every entry from TMDB on each run, including `slug`, `disc`, and `sale`.
**Why it's wrong:** Breaks shared deep links when TMDB corrects a title; wipes sold status and prices.
**Do this instead:** `enrich.ts` merges: metadata fields are replaced, `slug`/`id`/`disc`/`sale`/`ingest.addedAt` are preserved. Slug is written once.

### Anti-Pattern 6: Using `actions/configure-pages` `static_site_generator: next`

**What people do:** Copy the GitHub starter workflow, which injects `basePath = "/<repo>"` into `next.config`.
**Why it's wrong:** Silently overrides the env-driven `basePath`, so the build cannot be retargeted to `/sale/dvds` or `/`.
**Do this instead:** Mirror the sibling workflow (no configure-pages injection) and pass `NEXT_PUBLIC_BASE_PATH` explicitly.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| TMDB API v3 | Ingest-time only; Bearer token from env; `/search/movie`, `/movie/{id}?append_to_response=credits,external_ids`; `p-limit(4)`; disk cache | Soft ceiling ~40 req/s, 429 on excess (legacy 40/10 s limit removed Dec 2019). Attribution text + logo required in the site footer. Terms distinguish non-commercial (free) from commercial ("primary purpose is to create revenue") — a for-sale catalog is a grey area; PITFALLS/STACK should flag whether to email TMDB or rely on the personal-use reading. |
| TMDB image CDN (`image.tmdb.org/t/p/`) | Visitor's browser hotlinks; poster `w342` grid / `w500` detail; profile `w185` | Measured `cache-control: public, max-age=31536000`; 63–91 KB per w342 poster. Available sizes: posters w92/w154/w185/w342/w500/w780/original; profiles w45/w185/h632/original. |
| OMDb API | Ingest-time only; `?i=<imdbId>&apikey=…`; daily-budget guard | Free key: 1,000 req/day. Provides `imdbRating`, `imdbVotes`, `Ratings[]` (RT, Metacritic). Poster API is patron-only — never use OMDb for images. |
| IMDb | Outbound link only: `https://www.imdb.com/title/${imdbId}/` | No API, no scraping. |
| GitHub Pages | `actions/upload-pages-artifact@v3` from `out/` + `actions/deploy-pages@v4`; `public/.nojekyll`; `out/404.html` auto-served for unknown paths | Project site inherits `patrickclery.com` custom domain → `/dvd-seller/` for free. |
| Claude Code (skill host) | `.claude/skills/dvd-ingest/SKILL.md`; args via `$ARGUMENTS`; scripts via `${CLAUDE_PROJECT_DIR}/scripts/…`; `allowed-tools` pre-approves `Bash(npx tsx scripts/*)` and `Bash(git add data/catalog.json)`, `Bash(git commit *)` | Vision happens in-model by having the skill `Read` the photo files; no separate OCR service. |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| skill ↔ scripts | CLI + JSON files in a temp dir (`candidates.json` → `confirmed.json`) | Scripts are deterministic and testable without Claude. |
| scripts ↔ `data/catalog.json` | `enrich.ts`/`mark-sold.ts` are the only writers; both run `validate` before writing | Atomic write (write temp, rename). |
| `data/catalog.json` ↔ `src/lib/catalog.ts` | Static import at build; Zod parse fails the build on bad data | The build is the last line of defense against a bad skill run. |
| server components ↔ `CatalogBrowser` | Props (card projection) across the `'use client'` boundary | Never pass the full `Movie[]` to the client on the index page. |
| `src/lib/images.ts` ↔ everything rendering an image | Function call | Only module that knows about the CDN or `IMAGE_MODE`. |
| this repo ↔ `patrickclery.github.io` (topology b) | `repository_dispatch` event; main workflow checks out + builds this repo | One-way; the main site never needs to know about the data format. |

## Sources

Confidence tags per the `classify-confidence` seam (`webfetch` → LOW; official-doc claims below are cross-verified by at least two sources and treated as reliable for planning):

- Next.js `basePath` — https://nextjs.org/docs/app/api-reference/config/next-config-js/basePath (docs v16.3.5, updated 2025-06-16) — LOW (seam) / official
- Next.js static exports guide (trailingSlash layout, unsupported features, `generateStaticParams` requirement, `out/404.html`) — https://nextjs.org/docs/app/guides/static-exports (updated 2026-08-25) — LOW (seam) / official
- Next.js `generateStaticParams` + `dynamicParams` — https://nextjs.org/docs/app/api-reference/functions/generate-static-params (updated 2026-08-25) — LOW (seam) / official
- Next.js `useSearchParams` Suspense requirement and build error — https://nextjs.org/docs/app/api-reference/functions/use-search-params (updated 2026-07-14) — LOW (seam) / official
- Official GitHub Pages template: `basePath: process.env.PAGES_BASE_PATH` — https://github.com/nextjs/deploy-github-pages (`next.config.ts`, `.github/workflows/deploy.yml`) — LOW (seam) / official, cross-verifies the env-driven basePath pattern
- `actions/configure-pages` injects `output`, `basePath`, `images.unoptimized` for `next` — https://raw.githubusercontent.com/actions/configure-pages/main/src/set-pages-config.js — LOW (seam) / official source code
- GitHub Pages custom 404 (`404.html` at publishing root) — https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-custom-404-page-for-your-github-pages-site — LOW (seam) / official
- GitHub Pages custom domains: project sites inherit the user-site domain; per-repo CNAME subdomain override — https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages — LOW (seam) / official
- `.nojekyll` needed for `_next/` — community consensus (gregrickaby/nextjs-github-pages, Viget, dev.to) plus sibling repo already ships `public/.nojekyll` — LOW (websearch) / corroborated locally
- `actions/download-artifact` cross-repo inputs (`repository`, `run-id`, `github-token` with `actions:read`) — https://github.com/actions/download-artifact — LOW (seam) / official
- TMDB image configuration (`secure_base_url`, `poster_sizes`, `profile_sizes`) — https://developer.themoviedb.org/reference/configuration-details and https://developer.themoviedb.org/docs/image-basics — LOW (seam) / official
- TMDB rate limiting (~40 req/s soft ceiling, 429, legacy limit removed 2019) — https://developer.themoviedb.org/docs/rate-limiting — LOW (seam) / official
- TMDB attribution + commercial-use terms — https://developer.themoviedb.org/docs/faq — LOW (seam) / official
- TMDB `append_to_response` — https://developer.themoviedb.org/docs/append-to-response, https://developer.themoviedb.org/reference/movie-details — LOW (seam) / official
- Live measurement 2026-09-21: `curl -I https://image.tmdb.org/t/p/w342/<path>` → 63,663 / 77,336 / 90,922 bytes; `cache-control: public, max-age=31536000` — direct observation
- OMDb free tier 1,000 req/day — https://www.omdbapi.com/apikey.aspx — LOW (seam) / official
- Claude Code skills (project `.claude/skills/`, frontmatter, `${CLAUDE_SKILL_DIR}`, `${CLAUDE_PROJECT_DIR}`, `$ARGUMENTS`, `allowed-tools`) — https://code.claude.com/docs/en/skills — LOW (seam) / official
- Sibling repo conventions — `~/github.com/patrickclery/patrickclery.github.io/{next.config.ts,.github/workflows/deploy.yml,public/.nojekyll,public/CNAME}` — direct inspection

---
*Architecture research for: static DVD-collection catalog SPA with agent-driven ingestion*
*Researched: 2026-09-21*
