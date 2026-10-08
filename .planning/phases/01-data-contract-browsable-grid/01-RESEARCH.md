# Phase 1: Data Contract & Browsable Grid - Research

**Researched:** 2026-10-07
**Domain:** Next.js 16 static export (App Router) + zod 4 data contract + client-side catalog filtering, deployed under a configurable base path
**Confidence:** HIGH — every load-bearing claim below was exercised in a throwaway scaffold built with the exact pinned versions (`scratchpad/probe`, Next 16.3.5 / React 19.3.0 / TS 5.9.3 / Tailwind 4.3.3 / zod 4.6.5 / Vitest 5.0.3 / lucide-react 1.52.0) and cross-checked against the official docs fetched this session. Items that could not be exercised (TMDB response bodies — the secret-read guard blocks `--env-file=.env.local`) are tagged `[CITED]` or `[ASSUMED]`.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

#### Schema shape (DATA-01..05, DATA-07)
- **D-01:** Adopt the zod schema proposed in `.planning/research/ARCHITECTURE.md` § "Proposed `catalog.json` Entry Schema" as the starting point, with the tweaks below. — **Reversibility:** one-way — every entry the skill writes and every deep-link slug is derived from it; changing identity/sale fields later means re-ingesting hundreds of titles.
- **D-02:** `sale.status` enum is `available | reserved | sold`. Phase 1 UI treats `reserved` as available (shown under the Available filter) with a small "Reserved" badge; only `sold` gets the SOLD treatment. Rationale: adding an enum value later is a migration; rendering it is one badge.
- **D-03:** `disc.condition` uses eBay's buyer-familiar vocabulary: `new | like-new | very-good | good | acceptable` (replaces ARCHITECTURE's `fair | poor`). Display labels: "New", "Like New", "Very Good", "Good", "Acceptable".
- **D-04:** `disc.format` enum `DVD | Blu-ray | 4K UHD`, default `DVD`. `disc.region` free-text nullable. `disc.edition` free-text nullable; `(tmdbId, disc.edition)` is the uniqueness key.
- **D-05:** Money is `sale.priceCents` (integer) + `sale.currency` (ISO 4217, default `CAD`). Display via `Intl.NumberFormat("en-CA", { style: "currency", currency })`, dropping `.00` for whole amounts ("$12", "$12.50"). `priceCents: null` renders "Ask".
- **D-06:** Ratings object stores `imdb`, `imdbVotes`, `tmdb`, `tmdbVotes` (plus nullable `rottenTomatoes`, `metacritic` for later). The grid badge and the rating-threshold filter use `ratings.imdb ?? ratings.tmdb`; the badge is labelled by source ("IMDb 8.2" vs "TMDB 7.9") so a mixed catalog is never misleading.
- **D-07:** Image paths are stored TMDB-relative only (`posterPath: "/abc.jpg"`), never full URLs. `src/lib/images.ts` is the only module that builds `https://image.tmdb.org/t/p/{size}{path}` (grid `w342`, later detail `w500`, headshots `w185`). Reserve the `NEXT_PUBLIC_IMAGE_MODE=local|remote` switch in that module now (default `remote`); the local branch is v2.
- **D-08:** Slug format `^\d+-[a-z0-9-]+(--[a-z0-9-]+)?$` → `{tmdbId}-{kebab(title)}` with optional `--{kebab(edition)}` suffix for duplicate editions. Written once, never regenerated; `id === slug`. Lowercase only (GitHub Pages is case-sensitive).
- **D-09:** Catalog root: `{ version: 1, updatedAt, attribution: "This product uses the TMDB API but is not endorsed or certified by TMDB.", movies: [] }`. `movies` sorted by `tmdbId` ascending; object keys emitted in schema declaration order; 2-space indent; trailing newline. A no-op rewrite must produce an empty git diff (a `scripts/format-catalog.ts` or the validate script in `--write` mode owns this).
- **D-10:** Provenance block `ingest: { sourcePhoto, ocrTitle, confidence, confirmedBy: "auto"|"seller"|"seed", addedAt, enrichedAt }`. `enrichedAt` is the field that honors TMDB's 6-month cache rule (`fetchedAt` in REQUIREMENTS maps to this name). Seed entries use `confirmedBy: "seed"`.
- **D-11:** Genres stored as TMDB genre *names* (strings), not IDs, so the SPA needs no lookup table.
- **D-12:** Also export a JSON Schema (`data/catalog.schema.json`) generated from the zod schema so non-TypeScript tooling and the skill can validate; `$schema` pointer in `catalog.json` is optional.

#### Seed data (DATA-01, success criterion 1)
- **D-13:** Seed with ~10 real, well-known titles fetched from TMDB via a one-off `scripts/seed.ts` (plain `fetch`, `TMDB_API_TOKEN` v4 read token from `.env.local`). Include deliberate variety: at least one `sold`, one `reserved`, one with `priceCents: null` ("Ask"), one with no poster (placeholder path), one Blu-ray, titles spanning several decades and genres, one remake pair (e.g., The Thing 1982) so the year is exercised. — **Reversibility:** reversible — seed rows are replaced by real ingestion in Phase 4.
- **D-14:** **External dependency:** Patrick must create a free TMDB account and put `TMDB_API_TOKEN=<v4 read access token>` in `.env.local` before the seed task runs. The planner MUST make this an explicit checkpoint (`checkpoint:human-action`) rather than assuming the key exists. No key is ever committed or read from `NEXT_PUBLIC_*`.
- **D-15:** `scripts/seed.ts` is throwaway-grade but must write through the same zod-validated, deterministic writer that `validate --write` uses, so Phase 3's `enrich.ts` can replace it without touching the data file format. Do not build the throttled/cached TMDB client here — that is Phase 3.

#### Visual design & card layout (CAT-01, CAT-02, CAT-11, SALE-04)
- **D-16:** Inherit the sibling site's look so the two feel like one property when mounted: dark slate palette (`--color-bg #0F172A`, `--color-surface #1E293B`, `--color-border #334155`, `--color-text #F8FAFC`, `--color-text-muted #94A3B8`, `--color-accent #22C55E`, `--color-accent-warm #F59E0B`), Archivo for headings, Space Grotesk for body, `lucide-react` icons, Tailwind v4 CSS-first tokens in `globals.css`. Dark is the only theme (movie-store feel; no light mode work). — **Reversibility:** reversible — tokens live in one file.
- **D-17:** Card = poster in a fixed `aspect-[2/3]` box with `object-cover`, `loading="lazy"`, `decoding="async"`; caption BELOW the poster (title, year · rating badge, price chip + condition chip). No hover-only information (mobile has no hover); hover adds a subtle border-accent like the sibling's project cards. Whole card is a link (to the Phase 2 detail route `/movie/{slug}/`; in Phase 1 the link exists and may 404 until Phase 2).
- **D-18:** Missing poster → placeholder card: surface-colored 2:3 box with the title (and year) centered in Archivo, plus a faint `Film` icon. No external placeholder service.
- **D-19:** SOLD treatment: poster `grayscale` + reduced opacity, diagonal "SOLD" ribbon in `--color-accent-warm` across the top-left corner, price chip replaced by "Sold". Reserved: small warm-colored "Reserved" pill over the poster, no desaturation.
- **D-20:** Grid columns: 2 at ≥360px, 3 at `sm`, 4 at `md`, 5 at `lg`, 6 at `xl`; gap 3–4; container `max-w-7xl`. Touch targets ≥44px on all controls.

#### Filter/sort/search UI (CAT-03..CAT-10)
- **D-21:** Desktop (`md+`): a sticky top toolbar with search input (left), sort `<select>` (right), and a horizontally scrollable row of filter chips beneath (genre multi-select chips, decade chips, rating-threshold chips `Any | 6+ | 7+ | 8+`, availability segmented control `Available | Sold | All`). Mobile: same sticky toolbar with search + sort, plus a "Filters (n)" button that opens a bottom sheet containing the same chip groups and a "Show X results" apply button. One shared chip component; the sheet is just a different container.
- **D-22:** Sort options: `Popularity` (default, TMDB popularity desc), `Rating` (displayed rating desc, nulls last), `Year` (newest first), `Title` (A→Z). Sorting is stable (secondary key title).
- **D-23:** Search is client-side substring match on `title` and `originalTitle`, diacritics-stripped (`normalize("NFD")` + strip combining marks) and case-insensitive, applied on every keystroke with no debounce needed at a few hundred items. No Fuse.js in Phase 1.
- **D-24:** Result count line "Showing 42 of 318" sits above the grid with a "Clear all" link that appears only when any filter/search/sort deviates from default. Empty state: centered message "No matches" + "Clear filters" button.
- **D-25:** Default availability filter = `Available` (which includes `reserved`). Sold titles are hidden unless the visitor picks `Sold` or `All`.

#### URL state (CAT-09)
- **D-26:** Hand-rolled `URLSearchParams` (no `nuqs` dependency). Keys: `q`, `genre` (comma-separated kebab names), `decade` (comma-separated, e.g. `1990,2000`), `min` (rating threshold), `avail` (`sold|all`; omitted for default), `sort` (`rating|year|title`; omitted for default). Defaults are omitted so the canonical grid URL is bare. Updates use `router.replace` (no history spam per keystroke); back-navigation from a detail page restores the view because the state is in the URL.
- **D-27:** The component that reads `useSearchParams` lives inside a `<Suspense>` boundary in the server `page.tsx` from day one (build-time failure otherwise). Filter/sort/search logic is pure functions in `src/lib/filters.ts` with unit tests; the client component stays thin.

#### Base path & static export (DEP-02, CAT-13)
- **D-28:** `next.config.ts`: `output: "export"`, `trailingSlash: true`, `images: { unoptimized: true }`, `basePath` and `assetPrefix` from `process.env.NEXT_PUBLIC_BASE_PATH ?? ""`. `src/lib/base-path.ts` exports `withBasePath(path)`; every non-`next/link` URL (OG images later, manual `<a>`, any `fetch`) goes through it. TMDB CDN URLs are absolute and bypass it.
- **D-29:** Catalog is imported at build time (`import catalog from "@/../data/catalog.json"` parsed through zod in `src/lib/catalog.ts`); the index page passes a slim card projection (slug, title, year, posterPath, rating + source, genres, decade, status, priceCents, currency, condition, popularity) to the client component — never the full `Movie[]`.
- **D-30:** `public/.nojekyll` committed; CI-side checks (matrix build, link checker) are Phase 2.

#### Seller configuration (CAT-12)
- **D-31:** Seller-facing config lives in `data/seller.json` validated by a `Seller` zod schema in `src/lib/schema.ts`: `{ name, email, location, currency: "CAD", blurb, terms }` (blurb = one paragraph shown in the header; terms = pickup/shipping/payment one-liners). JSON, not TS, so other sellers can edit it without touching code. `email` is used by Phase 2's mailto. Phase 1 ships Patrick's real values (Montreal, CAD).
- **D-32:** Header shows: site title ("DVDs for Sale" — final wording is Claude's discretion), "N available · M total" counts computed at build, the seller blurb, and the TMDB attribution notice + logo in the footer of the root layout (DTL-08 is Phase 2, but the attribution footer is trivial and lands with the layout now).

#### Repo hygiene & tooling (DATA-06, DATA-08)
- **D-33:** Stack pinned per `.planning/research/STACK.md`: Next.js 16.x, React 19.x, TypeScript 5.9.x (not 7), Tailwind 4.x via `@tailwindcss/postcss`, zod 4.x, lucide-react, `tsx` for scripts, Node 22 (`.nvmrc` + `engines`). `npm` with committed lockfile. Vitest for `filters.ts` and schema tests.
- **D-34:** `package.json` scripts: `dev`, `build`, `prebuild` → `validate`, `validate` → `tsx scripts/validate.ts`, `seed` → `tsx scripts/seed.ts`, `test`. `.gitignore`: `.env*`, `.cache/`, `photos/`, `*.jpg|*.jpeg|*.png|*.heic` outside `public/`, `out/`, `.next/`, `node_modules/`.
- **D-35:** Conventions match the sibling repo: PascalCase `.tsx` components with `export function`, inline prop types, 2-space indent, trailing commas, `@/*` alias. Add Prettier + ESLint (next/core-web-vitals) since this repo will have scripts and tests — light config, no custom rules.

### Claude's Discretion
- Exact copy for the header title, empty-state text, and chip labels.
- Whether the bottom sheet is a hand-rolled `<dialog>` or a small headless component — no new heavy UI library.
- Skeleton/loading treatment for the Suspense fallback.
- Test runner details and file layout under `src/lib/__tests__/` vs co-located.
- Whether `scripts/validate.ts` and the deterministic writer are one file with flags or two modules.

### Deferred Ideas (OUT OF SCOPE)
- Actor/director search, "more in this collection", trailer/TMDB/Letterboxd links, recently-added row, box-set entry type, wishlist mailto, density toggle, `/` shortcut — all v2 in REQUIREMENTS.md; schema fields for cast/similar/videos are stored in Phase 1 so these are additive later.
- Local image cache mode — switch reserved in `images.ts` (D-07); implementation is v2 (IMG-01).
- Fuse.js fuzzy search — only if UAT shows misspellings matter.
- Light theme — not planned; dark only.
- Emailing TMDB about the commercial grey area — optional, recorded in PROJECT.md Key Decisions; not blocking.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DATA-01 | Single committed `data/catalog.json` validated by one shared zod schema (`src/lib/schema.ts`) imported by SPA and scripts | § JSON import at build time (probe: `import raw from "../../data/catalog.json"` + `Catalog.parse(raw)` works in a Server Component with `resolveJsonModule`); § Code Examples → `schema.ts`, `catalog.ts` |
| DATA-02 | Entry keyed by `tmdbId`, slug `{tmdbId}-{kebab-title}` generated once | § Code Examples → `slug()` helper; zod `.regex` on D-08 pattern verified to emit `pattern` in JSON Schema |
| DATA-03 | Entry stores TMDB metadata (title, year, runtime, genres, certification, tagline, overview, poster/backdrop, popularity, TMDB rating, imdbId, director, cast, trailer key, similar IDs) + optional `imdbRating` | § Seed script (field mapping from `/3/movie/{id}?append_to_response=credits,external_ids,videos,release_dates,similar`; US certification pick) |
| DATA-04 | Seller-owned fields separate from metadata (`disc`, `sale`) | § Code Examples → `schema.ts` (`disc`, `sale` sub-objects with D-02/D-03/D-04/D-05 enums) |
| DATA-05 | Provenance (`enrichedAt`, `sourcePhoto`, `confirmedBy`) | § Code Examples → `schema.ts` `ingest` block incl. `"seed"` |
| DATA-06 | `validate` runs as `prebuild`; malformed entry fails build loudly | § Deterministic writer + validate (probe: `prebuild` aborted `next build` with `✖ Invalid option: expected one of "new"\|"like-new"\|... → at movies[0].disc.condition`) |
| DATA-07 | Deterministic ordering/key order so diffs show only real changes | § Deterministic JSON writer (probe: `validate --write` twice → `cmp` identical; non-canonical file fails `validate` in check mode) |
| DATA-08 | `.gitignore` covers secrets/photos; no API key in client bundles | § Environment / Security (existing `.gitignore` verified; token read only in `scripts/seed.ts` via `tsx --env-file`; probe grep of `out/` for `TMDB_API_TOKEN`/`api.themoviedb.org` → none) |
| CAT-01 | Phone-first poster grid 2→6 columns with poster/title/year/rating | § Architecture Patterns → card projection; UI-SPEC grid classes; Tailwind `@theme inline` tokens verified to compile |
| CAT-02 | Lazy TMDB posters in fixed 2:3 box, title placeholder when none | § Code Examples → `images.ts`, `Poster` with `onError` fallback; TMDB image URL pattern `[CITED]` |
| CAT-03..CAT-07 | Genre / sort / rating threshold / decade / availability filters | § Client filtering (`filters.ts` pure functions, stable sort with title tiebreak, nulls-last rating) |
| CAT-08 | Instant diacritics-insensitive title search | § Client filtering (`normalize("NFD").replace(/\p{M}/gu,"")` verified on Node 22: `"Amélie Léon Ça"` → `"amelie leon ca"`) |
| CAT-09 | Filters/sort/search persisted in URL, survive back-nav | § Static export gotchas (`useSearchParams` + `<Suspense>` + `router.replace(..., { scroll: false })` — exact build error captured when Suspense is missing) |
| CAT-10 | Result count, clear-all, empty state | § Client filtering → `isDefault()` helper; UI-SPEC copy |
| CAT-11 | SOLD treatment when shown | UI-SPEC `SoldRibbon`; no new research needed |
| CAT-12 | Header counts + configurable seller blurb | § Code Examples → `seller.json` + `Seller` schema; counts computed in the Server Component |
| CAT-13 | Zero runtime API calls; only `image.tmdb.org` loads | § Architecture (build-time import; fonts self-hosted — probe `out/` has 0 references to `fonts.googleapis`/`fonts.gstatic`) |
| SALE-04 | Price + condition chips on cards | § Client filtering → `formatPrice()` (pitfall: `maximumFractionDigits: 2` yields "$12.5"; use `cents % 100 === 0 ? 0 : 2`) |
| DEP-02 | `NEXT_PUBLIC_BASE_PATH` configurable; every non-Link URL via `withBasePath()` | § Scaffold → `next.config.ts`; probe: `/sale/dvds` build emits only `/sale/dvds/...` hrefs/srcs, served from a nested folder → all 200 |
</phase_requirements>

## Summary

The stack in STACK.md is confirmed workable end-to-end, with three corrections that matter for the plan. (1) **Pin `next@16.3.5`, not `latest`**: `16.4.0` was published 2026-10-06 (one day before this research) and the legitimacy seam flags every "latest" release younger than a few days; 16.3.5 (2026-09-11) is the version the probe exercised. (2) **Pin `eslint@9.39.5` (npm `maintenance` tag), not ESLint 10**: `eslint-config-next@16.3.5` pulls `eslint-plugin-react@7.37.5`, whose peer range stops at `^9.7`, and under ESLint 10.12.0 it crashes with `contextOrFilename.getFilename is not a function` — reproduced in the probe; ESLint 9.39.5 runs clean. (3) **`next build` rewrites `tsconfig.json`** on first run (`jsx` → `react-jsx`, adds `.next/dev/types/**/*.ts` to `include`); commit those values up front so the first build does not dirty the tree.

Every static-export mechanism Phase 1 depends on was observed directly: `basePath` alone prefixes `_next/` chunks, fonts, and `next/link` hrefs (adding `assetPrefix: basePath` per D-28 is harmless — no double prefix); `public/.nojekyll` and `public/tmdb.svg` land in `out/`; `next/font/google` self-hosts Archivo + Space Grotesk as `/_next/static/media/*.woff2` (no Google requests at runtime); the `out/` folder served from `http://127.0.0.1:8765/sale/dvds/` via a symlink + `python3 -m http.server` returns 200 for every emitted asset; omitting `<Suspense>` around the `useSearchParams` consumer fails `next build` with `useSearchParams() should be wrapped in a suspense boundary at page "/"`; `next build` no longer lints. The static `index.html` contains the server-rendered header plus the Suspense fallback (`BAILOUT_TO_CLIENT_SIDE_RENDERING`), and the card projection rides in the RSC payload — so the skeleton in UI-SPEC is what first paints.

Data-contract tooling also checks out: `zod@4.6.5` exposes `z.toJSONSchema(schema, { target: "draft-2020-12", io: "input" })` and `z.prettifyError`; key order for the deterministic writer comes straight from `Object.keys(Movie.shape)`; `vitest@5.0.3` resolves the `@/*` alias with Vite 8's built-in `resolve: { tsconfigPaths: true }` (no `vite-tsconfig-paths` plugin); `tsx --env-file=.env.local scripts/seed.ts` loads the token into `process.env` without `dotenv`.

**Primary recommendation:** Scaffold by hand from the `package.json` below (exact pins), copy the sibling's `tsconfig`/`postcss`/`globals.css`/`layout.tsx` idioms from PATTERNS.md, and wire `prebuild → validate` before writing any UI so the data guard exists from the first commit.

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Schema definition + validation (`schema.ts`, `validate.ts`) | Build / Node scripts | — | Runs in `prebuild` and in Phase 3 scripts; never ships to the browser |
| Catalog import + card projection | Frontend server (SSG at `next build`) | — | Server Component reads JSON once; emits slim props into the RSC payload |
| Header counts, seller blurb, attribution footer | Frontend server (SSG) | — | Pure build-time data; prerendered into HTML |
| Filter / sort / search state | Browser / Client | — | URL is the store; `useSearchParams` + `router.replace`; pure functions in `filters.ts` |
| Bottom sheet, chips, result count | Browser / Client | — | Interactive; native `<dialog>`; no library |
| Poster delivery | CDN (`image.tmdb.org`) | Browser (`onError` → placeholder) | Hotlinked from the visitor's browser; site origin never calls TMDB |
| Fonts | CDN / Static (`out/_next/static/media`) | — | `next/font/google` self-hosts at build |
| Base path resolution | Build (inlined) | Browser (`withBasePath()` for `<img src>`) | `basePath` inlined into bundles; public assets need the manual prefix |
| Seed data fetch (TMDB) | Node script (`scripts/seed.ts`) | — | One-off, offline, token from `.env.local`; nothing in the SPA |

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `next` | **16.3.5** (exact pin) | Static export, App Router, `next/font` | Probe-verified build/export/basePath. `16.4.0` is 1 day old `[VERIFIED: npm registry — published 2026-10-06T18:35Z]`; stay on 16.3.5 until it ages. Engines `node >=20.9.0` `[VERIFIED: npm view next engines]` |
| `react`, `react-dom` | 19.3.0 | UI | Next peer `^19.0.0` `[VERIFIED: npm view next peerDependencies]` |
| `typescript` | 5.9.3 | Types | `latest` tag is 7.0.2 (Go-native) — do not use; `typescript@5` resolves 5.9.3 `[VERIFIED: npm dist-tags]` |
| `tailwindcss` + `@tailwindcss/postcss` | 4.3.3 | Styling, CSS-first tokens | Same version line required; `@tailwindcss/postcss@4.3.3` depends on `tailwindcss@4.3.3` exactly `[VERIFIED: npm view dependencies]` |
| `zod` | 4.6.5 | Schema + JSON Schema export | `z.toJSONSchema`, `z.prettifyError`, `.shape` all exercised in probe `[VERIFIED: probe]` |
| `lucide-react` | 1.52.0 | Icons | Renders `aria-hidden="true"`, `width/height=24`, `stroke-width=2`, `class="lucide lucide-film"` by default `[VERIFIED: probe renderToStaticMarkup]`; peer `react ^19` `[VERIFIED: npm]` |

### Supporting (devDependencies)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `tsx` | 4.23.15 | Run `scripts/*.ts`; passes `--env-file` through to Node | `validate`, `seed`; `[VERIFIED: probe — `tsx --env-file=<file> script.ts` populated `process.env`]` |
| `vitest` | 5.0.3 | Unit tests for `filters.ts`, `schema.ts` | Requires Node ≥ 22.12 and Vite ≥ 6.4 `[CITED: vitest.dev/guide/migration]`; bundles Vite 8.3.3 `[VERIFIED: probe]`. No `@vitejs/plugin-react` needed for node-environment lib tests |
| `eslint` | **9.39.5** (`maintenance` tag) | Lint | ESLint 10 crashes `eslint-plugin-react@7.37.5` (see Pitfall 2) `[VERIFIED: probe]` |
| `eslint-config-next` | 16.3.5 | Next/React/TS flat configs | Match the `next` pin; peer `eslint >=9.0.0` `[VERIFIED: npm]` |
| `prettier` | 3.9.9 | Format | Light `.prettierrc`; optionally `eslint-config-prettier@10.1.8` (peer `eslint >=7`) `[VERIFIED: npm]` |
| `@types/node` | 22.20.5 | Node 22 types | Latest 22.x line `[VERIFIED: npm view @types/node@22]` |
| `@types/react`, `@types/react-dom` | 19.3.0 | React types | `[VERIFIED: npm]` |

Not needed: `dotenv` (Node `--env-file` suffices), `vite-tsconfig-paths` (Vite 8 has `resolve.tsconfigPaths`), `p-throttle` (Phase 3), `nuqs` (D-26 forbids), `@vitejs/plugin-react` (no component tests in Phase 1).

### Alternatives Considered
Per the ROADMAP ("Research: no") and CONTEXT.md, alternatives were not surveyed. The only version choices made here are *within* the locked stack (16.3.5 vs 16.4.0; ESLint 9 vs 10).

**Installation (exact pins; `npm install` then commit `package-lock.json`, lockfileVersion 3):**
```json
{
  "name": "dvd-seller",
  "version": "0.1.0",
  "private": true,
  "engines": { "node": ">=22" },
  "scripts": {
    "dev": "next dev",
    "prebuild": "npm run validate",
    "build": "next build",
    "validate": "tsx scripts/validate.ts",
    "seed": "tsx --env-file-if-exists=.env.local scripts/seed.ts",
    "test": "vitest run",
    "lint": "eslint .",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  },
  "dependencies": {
    "lucide-react": "1.52.0",
    "next": "16.3.5",
    "react": "19.3.0",
    "react-dom": "19.3.0",
    "zod": "4.6.5"
  },
  "devDependencies": {
    "@tailwindcss/postcss": "4.3.3",
    "@types/node": "22.20.5",
    "@types/react": "19.3.0",
    "@types/react-dom": "19.3.0",
    "eslint": "9.39.5",
    "eslint-config-next": "16.3.5",
    "prettier": "3.9.9",
    "tailwindcss": "4.3.3",
    "tsx": "4.23.15",
    "typescript": "5.9.3",
    "vitest": "5.0.3"
  }
}
```
`[VERIFIED: probe — this exact manifest (minus `seed`/`format*`) installed 387 packages in 17 s, built, tested, and linted (with eslint 9.39.5)]`. Do **not** add `"type": "module"` — the probe used it and it is unnecessary; Next's `next.config.ts` and `postcss.config.mjs` work without it, and omitting it keeps parity with the sibling. Do not use `create-next-app`: it would scaffold `eslint@10`-era defaults and a `README`/`app/` layout that differ from the sibling.

**Version verification performed:** `npm view <pkg> version|dist-tags|peerDependencies|time` on 2026-10-07 for every package above.

## Package Legitimacy Audit

Seam: `gsd-tools query package-legitimacy check --ecosystem npm …` (2026-10-07). Every `SUS` verdict below carries the single reason `too-new`, meaning the package's *latest release* is only days old — not that the package is unknown. All are top-1000 npm packages with tens of millions of weekly downloads; the mitigation is to **pin a release that has aged** (done above) rather than a checkpoint per install. No `postinstall` scripts on any of them `[VERIFIED: npm view <pkg> scripts.postinstall → empty for next, tsx, vitest, eslint-config-next, zod, lucide-react, tailwindcss, @tailwindcss/postcss, prettier]`.

| Package | Registry | Latest published | Downloads/wk | Source Repo | Verdict | Disposition |
|---------|----------|------------------|--------------|-------------|---------|-------------|
| next | npm | 16.4.0 on 2026-10-06 | 76.9M | github.com/vercel/next.js | SUS (too-new) | Approved — **pin 16.3.5** (2026-09-11) |
| react / react-dom | npm | 19.3.0 | 224M / 212M | github.com/facebook/react | SUS (too-new) | Approved — pin 19.3.0 (Next 16 peer) |
| typescript | npm | 7.0.2 (`latest`) | 365M | github.com/microsoft/TypeScript | OK | Approved — pin **5.9.3** |
| tailwindcss | npm | 4.3.3 (2026-07-16) | 163M | github.com/tailwindlabs/tailwindcss | OK | Approved |
| @tailwindcss/postcss | npm | 4.3.3 | 49.6M | same | OK | Approved |
| zod | npm | 4.6.5 (2026-09-13) | 387M | github.com/colinhacks/zod | SUS (too-new) | Approved — 4.6.5 is 3+ weeks old |
| lucide-react | npm | 1.52.0 (2026-10-04) | 135M | github.com/lucide-icons/lucide | SUS (too-new) | Approved — 1.x line since 2026-03-23; probe-verified |
| vitest | npm | 5.0.3 (2026-09-30) | 142M | github.com/vitest-dev/vitest | SUS (too-new) | Approved — probe-verified; fallback `4.1.11` (`V4` tag) if any issue |
| tsx | npm | 4.23.15 (2026-09-20) | 115M | github.com/privatenumber/tsx | SUS (too-new) | Approved |
| eslint | npm | 10.12.0 (2026-10-02) | 198M | github.com/eslint/eslint | SUS (too-new) | Approved — **pin 9.39.5** (`maintenance`) for plugin compat |
| eslint-config-next | npm | 16.4.0 | 42.4M | github.com/vercel/next.js | SUS (too-new) | Approved — pin 16.3.5 |
| prettier | npm | 3.9.9 (2026-09-23) | 169M | github.com/prettier/prettier | SUS (too-new) | Approved |
| @types/node, @types/react, @types/react-dom | npm | — | 549M / 205M / 176M | DefinitelyTyped | SUS (too-new) | Approved — pin 22.20.5 / 19.3.0 / 19.3.0 |
| serve (optional, local nested-prefix test) | npm | 14.2.6 (2026-03-03) | 4.9M | github.com/vercel/serve | OK | Not required — `python3 -m http.server` recipe below needs no install |

**Packages removed due to [SLOP] verdict:** none
**Packages flagged as suspicious [SUS]:** all `too-new` on *latest*; resolved by pinning aged releases. The planner does not need a `checkpoint:human-verify` per install — one checkpoint after `npm install` to eyeball `package-lock.json` is sufficient.

## Architecture Patterns

### System Architecture Diagram

```
                       BUILD TIME (next build, Node 22)                          RUNTIME (visitor browser)
 ┌────────────────────────────────────────────────────────────────┐   ┌──────────────────────────────────────┐
 │ data/catalog.json ──┐                                          │   │  GET /sale/dvds/  (GitHub Pages)      │
 │ data/seller.json  ──┤  prebuild: tsx scripts/validate.ts       │   │   └─ index.html = header + skeleton  │
 │                     │   ├─ Catalog.safeParse / Seller.parse     │   │       + RSC payload (card projection)│
 │ src/lib/schema.ts ──┘   ├─ uniqueness: slug, (tmdbId,edition)  │   │            │ hydrate                 │
 │        │                ├─ canonical-form check (fails if ≠)   │   │            ▼                         │
 │        │                └─ writes data/catalog.schema.json     │   │  <CatalogBrowser>  (inside Suspense) │
 │        ▼                                                        │   │   useSearchParams ─► parseParams()   │
 │ src/lib/catalog.ts  Catalog.parse(raw) ─► getCards() projection│   │        │                             │
 │        │                                                        │   │        ▼                             │
 │        ▼                                                        │   │   applyFilters(cards, state)  pure   │
 │ src/app/page.tsx (Server Component)                             │   │   sortCards(...)             pure    │
 │   <Header counts blurb/>                                        │   │        │                             │
 │   <Suspense fallback={<GridSkeleton/>}>                         │   │        ▼                             │
 │     <CatalogBrowser cards={cards}/>   ──► RSC payload           │   │   <ul grid> <MovieCard> ×N           │
 │   </Suspense>                                                   │   │     <img src=image.tmdb.org/t/p/w342…│──► TMDB CDN
 │   <Footer attribution + tmdb.svg via withBasePath()/>           │   │     onError ─► <PosterPlaceholder>   │
 │        │                                                        │   │                                      │
 │        ▼  output:"export", trailingSlash, basePath (inlined)    │   │   chip/search/sort change            │
 │ out/index.html, out/404.html, out/_next/**, out/.nojekyll,      │   │     └─► router.replace(path?qs,      │
 │ out/tmdb.svg                                                    │   │              { scroll:false })        │
 └────────────────────────────────────────────────────────────────┘   └──────────────────────────────────────┘
 Seed (one-off, offline): tsx --env-file=.env.local scripts/seed.ts ─► TMDB v3 API ─► same writer as validate --write
```

### Recommended Project Structure
```
dvd-seller/
├── .nvmrc                      # 22
├── .prettierrc                 # { "semi": true, "singleQuote": false, "trailingComma": "all" }
├── eslint.config.mjs           # defineConfig([...nextVitals, ...nextTs, globalIgnores([...])])
├── next.config.ts              # output export, trailingSlash, basePath/assetPrefix from env, images.unoptimized
├── postcss.config.mjs          # @tailwindcss/postcss
├── tsconfig.json               # sibling + target ES2022 + jsx react-jsx + .next/dev/types include
├── vitest.config.ts            # resolve.tsconfigPaths, environment node
├── data/
│   ├── catalog.json            # canonical form (validate --write)
│   ├── catalog.schema.json     # generated by validate (z.toJSONSchema)
│   └── seller.json
├── public/
│   ├── .nojekyll               # zero bytes — copied to out/ verbatim
│   └── tmdb.svg                # official "primary short" blue logo
├── scripts/
│   ├── validate.ts             # check | --write ; emits schema json
│   ├── seed.ts                 # one-off TMDB fetch → writeCatalog()
│   └── lib/catalog-io.ts       # readCatalog / canonicalize / writeCatalog (shared by validate + seed; Phase 3 reuses)
└── src/
    ├── app/{layout.tsx,page.tsx,globals.css}
    ├── components/{Header,Footer,CatalogBrowser,Toolbar,FilterChips,FilterSheet,MovieCard,Poster,
    │               PosterPlaceholder,RatingBadge,PriceChip,ConditionChip,SoldRibbon,ReservedPill,
    │               EmptyState,GridSkeleton}.tsx
    └── lib/{schema,catalog,seller,filters,url-state,base-path,images,format}.ts + __tests__/
```

### Pattern 1: `next.config.ts` for export under a configurable base path
**What:** Read `NEXT_PUBLIC_BASE_PATH` once; strip a trailing slash; pass `undefined` when empty (Next rejects `""` for `basePath`? — it accepts `""` as default, but `undefined` is the documented "unset"). `assetPrefix: basePath` is redundant (basePath already prefixes `_next/`) but harmless — D-28 asks for it.
**Verified:** `[VERIFIED: probe — with and without assetPrefix, `/sale/dvds` build emitted only `/sale/dvds/...` URLs; `grep -c "/sale/dvds/sale/dvds"` → 0]`; `[CITED: nextjs.org/docs/app/api-reference/config/next-config-js/assetPrefix — "We do not suggest you use a custom Asset Prefix for this use case" (sub-path hosting)]`.
```typescript
// next.config.ts
import type { NextConfig } from "next";

const basePath = process.env.NEXT_PUBLIC_BASE_PATH?.replace(/\/+$/, "") || undefined;

const nextConfig: NextConfig = {
  output: "export",
  trailingSlash: true,
  basePath,
  assetPrefix: basePath, // D-28; no-op relative to basePath (verified: no double prefix)
  images: { unoptimized: true },
};

export default nextConfig;
```
`NEXT_PUBLIC_*` vars are inlined into client bundles at build `[CITED: nextjs.org basePath — "value is inlined in the client-side bundles"]`, so `src/lib/base-path.ts` can read `process.env.NEXT_PUBLIC_BASE_PATH` in client code and get the same string the config used.

### Pattern 2: `tsconfig.json` as Next 16 will leave it
**What:** Commit the post-build shape so the first `next build` does not modify a tracked file.
**Verified:** `[VERIFIED: probe — build log: "The following mandatory changes were made to your tsconfig.json: jsx was set to react-jsx"; "include was updated to add '.next/dev/types/**/*.ts'"]`
```json
{
  "compilerOptions": {
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./src/*"] },
    "target": "ES2022"
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts", ".next/dev/types/**/*.ts"],
  "exclude": ["node_modules", "out"]
}
```
`resolveJsonModule: true` is all that is needed for `import raw from "../../data/catalog.json"` — no `with { type: "json" }` import attribute `[VERIFIED: probe — Turbopack build and `tsc --noEmit` both clean]`. Use the relative path from `src/lib/catalog.ts` (the `@/*` alias maps to `./src/*`, so `@/../data/...` is not expressible cleanly; D-29's wording is satisfied by the relative import).

### Pattern 3: Suspense-wrapped URL state (D-26/D-27)
**What:** `page.tsx` stays a Server Component; the one client component that calls `useSearchParams` sits in `<Suspense fallback={<GridSkeleton/>}>`. Writes go through `router.replace(href, { scroll: false })`.
**Verified:** `[VERIFIED: probe — removing Suspense fails the build: `⨯ useSearchParams() should be wrapped in a suspense boundary at page "/". Read more: https://nextjs.org/docs/messages/missing-suspense-with-csr-bailout` … `Export encountered an error on /page: /, exiting the build.`]`; `[CITED: nextjs.org/docs/app/api-reference/functions/use-router — `router.replace(href, { scroll: boolean, transitionTypes })`]`; `[CITED: use-search-params — "In development, routes are rendered on-demand, so useSearchParams doesn't suspend and things may appear to work without Suspense"]`.
```tsx
// src/app/page.tsx (Server Component)
import { Suspense } from "react";
import { CatalogBrowser } from "@/components/CatalogBrowser";
import { GridSkeleton } from "@/components/GridSkeleton";
import { Header } from "@/components/Header";
import { getCards, getCounts } from "@/lib/catalog";
import { seller } from "@/lib/seller";

export default function Home() {
  const cards = getCards();
  return (
    <main className="flex-1" aria-busy={undefined}>
      <Header counts={getCounts()} blurb={seller.blurb} />
      <Suspense fallback={<GridSkeleton />}>
        <CatalogBrowser cards={cards} total={cards.length} />
      </Suspense>
    </main>
  );
}
```
```tsx
// src/lib/url-state.ts (client hook; thin)
"use client";
import { usePathname, useRouter, useSearchParams } from "next/navigation";
import { parseParams, serializeParams, type FilterState } from "@/lib/filters";

export function useCatalogParams() {
  const router = useRouter();
  const pathname = usePathname();
  const sp = useSearchParams();
  const state = parseParams(sp);                       // pure
  const setState = (next: FilterState) => {
    const qs = serializeParams(next);                   // pure; omits defaults (D-26)
    router.replace(qs ? `${pathname}?${qs}` : pathname, { scroll: false });
  };
  return [state, setState] as const;
}
```
`usePathname()` returns the path **without** basePath and `router.replace` re-applies it — this is how `next/link`/router work per docs `[CITED: basePath docs — "When linking to other pages using next/link and next/router the basePath will be automatically applied"]`; the App Router `useRouter` behaving identically is `[ASSUMED]` (A1) and is cheap to confirm in the nested-serve UAT below.

### Pattern 4: Build-time import + projection (D-29)
`[VERIFIED: probe — static `index.html` is 7.2 kB with header prerendered, `<template data-dgst="BAILOUT_TO_CLIENT_SIDE_RENDERING">` + fallback where the grid goes, and the card slug present 3× (RSC payload + link)]`. The projection keeps that payload small; never pass `Movie[]`.

### Pattern 5: Tailwind v4 tokens — `:root` + `@theme inline` (UI-SPEC default #13)
**What:** Keep the sibling's `:root` block verbatim (so `bg-[var(--color-surface)]` works) and register the same variables in `@theme inline` so `bg-surface`, `text-text-muted`, `font-heading` utilities also exist.
**Verified:** `[VERIFIED: probe — `@theme inline { --color-bg: var(--color-bg); … --font-heading: var(--font-archivo); }` compiled; `bg-bg text-text font-heading` classes present in output CSS]`; `[CITED: tailwindcss.com/docs/theme — "Use @theme inline when referencing other CSS variables"]`.
```css
/* src/app/globals.css */
@import "tailwindcss";

@layer base {
  :root { /* sibling tokens verbatim — see PATTERNS.md */ }
  /* …sibling html/body/::selection/focus-visible rules verbatim… */
}

@theme inline {
  --color-bg: var(--color-bg);
  --color-surface: var(--color-surface);
  --color-border: var(--color-border);
  --color-text: var(--color-text);
  --color-text-muted: var(--color-text-muted);
  --color-accent: var(--color-accent);
  --color-accent-warm: var(--color-accent-warm);
  --font-heading: var(--font-archivo);
  --font-body: var(--font-space-grotesk);
}
```
Caveat `[ASSUMED]` (A2): defining `--color-bg` inside `@theme inline` with the same name as the `:root` variable is self-referential at the CSS-variable level; the probe compiled and the utilities resolve because `inline` emits `background-color: var(--color-bg)` which reads the `:root` value at runtime. If a visual check shows a missing color, rename the theme keys (e.g. `--color-ink: var(--color-text)`) — the sibling arbitrary-value idiom keeps working regardless.

### Pattern 6: Native `<dialog>` bottom sheet (Claude's discretion → recommended)
- `showModal()` gives top-layer, inert background, focus trap, `::backdrop`, and Esc→`cancel`→`close` for free `[CITED: MDN HTMLDialogElement/showModal]`. `<dialog>` is Baseline since March 2022; Safari/iOS ≥ 15.4; 97.1% global `[CITED: caniuse.com/dialog]`.
- **Do not rely on `closedby="any"` for backdrop (light) dismiss** — unsupported in Safari through 27.2 (73% global) `[CITED: caniuse.com/mdn-html_elements_dialog_closedby]`. Implement backdrop click manually: `onClick={(e) => { if (e.target === dialogRef.current) dialogRef.current.close(); }}` (clicks on the padding-less `<dialog>` element itself are backdrop clicks when the inner panel fills it).
- React 19 pattern: `const ref = useRef<HTMLDialogElement>(null)`; `useEffect(() => { open ? ref.current?.showModal() : ref.current?.close(); }, [open])`; listen to `onClose` to sync `open=false` (covers Esc). Body scroll lock: toggle `overflow-hidden` on `document.documentElement` in the same effect (Safari still scrolls the page behind a modal dialog) `[ASSUMED]` (A3 — behaviour, not API).
- Draft/commit model and copy are fully specified in UI-SPEC § Bottom sheet; no further research needed.

### Anti-Patterns to Avoid
- **`useSearchParams` outside Suspense** — passes `next dev`, fails `next build` (verified message above).
- **Hard-coding `/sale/dvds` or `/dvd-seller` anywhere** — basePath is inlined; only `NEXT_PUBLIC_BASE_PATH` may know it.
- **`<img src="/tmdb.svg">` without `withBasePath()`** — `public/` assets are *not* prefixed `[CITED: assetPrefix docs — "does not influence… files in the public folder"]`; verified in probe that the footer `<img>` needed the helper.
- **`"type": "module"` in `package.json`** — unnecessary; sibling omits it.
- **ESLint 10** — see Pitfall 2.
- **`maximumFractionDigits: 2` alone for prices** — see Pitfall 4.
- **Reading the token via `NEXT_PUBLIC_*` or importing `scripts/` from `src/`** — would inline the secret.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| JSON Schema for the skill | A hand-written `catalog.schema.json` | `z.toJSONSchema(Catalog, { target: "draft-2020-12", io: "input" })` | Verified output: `const` for literals, `pattern` for regex, `default` for `.default()` (3 emitted), `integer` for `.int()` `[VERIFIED: probe]` |
| Error messages for bad catalog rows | Custom issue walker | `z.prettifyError(result.error)` | Verified output: `✖ Invalid option: expected one of "new"\|"like-new"\|"very-good"\|"good"\|"acceptable"` / `→ at movies[0].disc.condition` |
| Key ordering | A separate "field order" list | `Object.keys(Movie.shape)` / `Object.keys(Catalog.shape)` | zod 4 preserves declaration order in `.shape` `[VERIFIED: probe — printed `id,slug,tmdbId,title`]`; one source of truth |
| tsconfig path aliases in tests | `vite-tsconfig-paths` plugin | `resolve: { tsconfigPaths: true }` in `vitest.config.ts` | Built into Vite 8 `[CITED: vite.dev/config/shared-options]`; `@/lib/filters` import resolved in probe `[VERIFIED]` |
| `.env` loading in scripts | `dotenv` | `tsx --env-file=.env.local` / `--env-file-if-exists` (Node ≥ 22.9) | `[VERIFIED: probe]`; `[CITED: nodejs.org v22 CLI docs — env var in environment takes precedence over file; `--env-file-if-exists` does not throw when missing]` |
| Currency formatting | String concat | `Intl.NumberFormat("en-CA", { style: "currency", currency, minimumFractionDigits: f, maximumFractionDigits: f })` with `f = cents % 100 ? 2 : 0` | Verified `$12`, `$12.50`-equivalent, `US$12` for USD |
| Diacritic folding | Lookup table | `s.normalize("NFD").replace(/\p{M}/gu, "").toLowerCase()` | Verified on Node 22 (`"Amélie Léon Ça"` → `"amelie leon ca"`); same code runs in browsers (ES2018 Unicode property escapes) |
| Modal/focus trap | Headless UI lib | native `<dialog>.showModal()` | See Pattern 6 |
| Font hosting | `<link>` to Google Fonts | `next/font/google` | Self-hosts `.woff2` in `out/_next/static/media`; 0 Google references in output `[VERIFIED: probe]` |

**Key insight:** every "infrastructure" need in this phase is covered by a platform or already-installed primitive; the only bespoke code is the schema, the pure filter functions, and the components.

## Common Pitfalls

### Pitfall 1: Pinning `next@latest` picks up a one-day-old release
**What goes wrong:** `npm i next` today resolves 16.4.0 (published 2026-10-06T18:35Z); STACK.md and this probe were validated on 16.3.5.
**How to avoid:** Exact pin `"next": "16.3.5"`, `"eslint-config-next": "16.3.5"`. Revisit in Phase 2.
**Warning signs:** `npm ls next` shows 16.4.x.

### Pitfall 2: ESLint 10 crashes `eslint-config-next`'s React plugin
**What goes wrong:** `npx eslint .` with `eslint@10.12.0` + `eslint-config-next@16.3.5` aborts: `TypeError: Error while loading rule 'react/display-name': contextOrFilename.getFilename is not a function` (from `eslint-plugin-react@7.37.5`, peer `eslint … ^9.7`) `[VERIFIED: probe]`. Next's docs claim ESLint 10 support but warn "Some of the plugins included in eslint-config-next don't list ESLint 10 in their peer dependencies yet" `[CITED: nextjs.org/docs/app/api-reference/config/eslint]`.
**How to avoid:** `"eslint": "9.39.5"` (npm `maintenance` tag). Verified clean run (one legitimate `no-unused-expressions` warning on a ternary-as-statement).
**Config (flat, verified):**
```js
// eslint.config.mjs
import { defineConfig, globalIgnores } from "eslint/config";
import nextVitals from "eslint-config-next/core-web-vitals";
import nextTs from "eslint-config-next/typescript";

export default defineConfig([
  ...nextVitals,
  ...nextTs,
  { rules: { "@next/next/no-img-element": "off" } }, // plain <img> is the locked choice (D-17)
  globalIgnores([".next/**", "out/**", "next-env.d.ts"]),
]);
```
`next lint` is removed in 16; `next build` does not lint (0 lint lines in build output) `[VERIFIED: probe]` `[CITED: eslint docs — "Starting with Next.js 16, next lint is removed"]`. Add `eslint-config-prettier@10.1.8` (`import prettier from "eslint-config-prettier/flat"`) only if rule/format conflicts appear.

### Pitfall 3: First `next build` rewrites `tsconfig.json`
**What goes wrong:** Sibling's `"jsx": "preserve"` is force-changed to `"react-jsx"`, and `.next/dev/types/**/*.ts` is appended to `include` — a dirty tree mid-task. `[VERIFIED: probe]`
**How to avoid:** Commit Pattern 2's `tsconfig.json` in the scaffold task.

### Pitfall 4: `Intl` drops `.50` with `maximumFractionDigits: 2`
**What goes wrong:** `{ minimumFractionDigits: 0, maximumFractionDigits: 2 }` formats 1250¢ as `$12.5`, violating D-05 ("$12.50") `[VERIFIED: Node 22]`. UI-SPEC's PriceChip spec copies the faulty option set.
**How to avoid:** `const f = cents % 100 === 0 ? 0 : 2; new Intl.NumberFormat("en-CA", { style: "currency", currency, minimumFractionDigits: f, maximumFractionDigits: f }).format(cents / 100)`. Unit-test 1200 → `$12`, 1250 → `$12.50`, 1299 → `$12.99`, USD → `US$12`.

### Pitfall 5: The deterministic writer must be the *only* writer, and check mode must compare bytes
**What goes wrong:** If `seed.ts` writes with its own `JSON.stringify(obj, null, 2)` the key order follows insertion order, not schema order, and the next `validate` fails ("not in canonical form") — exactly what happened in the probe when the hand-typed seed file was built. `[VERIFIED: probe]`
**How to avoid:** One `scripts/lib/catalog-io.ts` exporting `canonicalize(catalog)` (sort by `tmdbId`, then `disc.edition ?? ""`; order keys from `.shape` recursively for nested objects `ratings/disc/sale/ingest`; `JSON.stringify(..., null, 2) + "\n"`) and `writeCatalog()`. `validate` default mode: `canonicalize(parsed) === fs.readFileSync(path, "utf8")` else exit 1 with "run `npm run validate -- --write`". `--write` mode: write and continue. Idempotency verified: second `--write` → `cmp` identical.
**Note on `.default()`:** `Catalog.parse` fills defaults (`format: "DVD"`, `status: "available"`, `currency: "CAD"`), so canonical output always contains them explicitly — good (diff-stable); use `io: "input"` for the JSON Schema so the skill may omit them.

### Pitfall 6: `useSearchParams` works in dev, fails the export
Covered in Pattern 3; the probe captured the exact build error. Also wrap any *future* consumer (Phase 2 "back to grid" link) the same way.

### Pitfall 7: Public assets and `basePath`
`out/tmdb.svg` exists, but `<img src="/tmdb.svg">` 404s under `/sale/dvds/`. Every `public/` reference goes through `withBasePath()`; `next/link` hrefs do not `[VERIFIED: probe — `/sale/dvds/movie/1091-the-thing/` emitted by `<Link href="/movie/…/">`]`.

### Pitfall 8: Lucide v1 icons are `aria-hidden` by default
Good for decorative icons, but icon-only buttons (close X, Filters) need `aria-label` on the `<button>`, not the icon `[CITED: lucide.dev/guide/advanced/accessibility — "Lucide icons are hidden from screen readers by default with aria-hidden='true'"]`. v1 also removed UMD builds and brand icons `[CITED: github.com/lucide-icons/lucide/releases/tag/1.0.1]`; the seven icons UI-SPEC uses (`Film, Search, SearchX, SlidersHorizontal, X, ArrowUpDown, Star`) all exist in 1.52.0 `[VERIFIED: probe import]`.

### Pitfall 9: Secret-read guard in this environment
A Claude Code hook blocks any Bash command whose text references `.env.local` (including `--env-file=.env.local`). The executor can still run `npm run seed` because the file name lives inside `package.json`'s script, not the command line — but if the hook also inspects child commands, fall back to asking Patrick to run `npm run seed` himself (planner: make the seed task a `checkpoint:human-action` as D-14 already requires). `[VERIFIED: this session — both `grep … .env.local` and `node --env-file=.env.local …` were blocked]`

## Code Examples

### `src/lib/schema.ts` (D-01..D-12, D-31; zod 4)
```typescript
import { z } from "zod";

const tmdbPath = z.string().regex(/^\/[\w-]+\.(jpg|png)$/);
const isoDate = z.string(); // ISO 8601; tighten in Phase 3 if needed

export const CastMember = z.object({
  tmdbId: z.number().int(),
  name: z.string(),
  character: z.string().nullable(),
  profilePath: tmdbPath.nullable(),
  order: z.number().int(),
});

export const Movie = z.object({
  id: z.string(),
  slug: z.string().regex(/^\d+-[a-z0-9-]+(--[a-z0-9-]+)?$/),
  tmdbId: z.number().int(),
  imdbId: z.string().regex(/^tt\d{7,8}$/).nullable(),
  title: z.string(),
  originalTitle: z.string().nullable(),
  year: z.number().int().nullable(),
  releaseDate: isoDate.nullable(),
  runtimeMinutes: z.number().int().nullable(),
  certification: z.string().nullable(),          // US cert from release_dates (DATA-03)
  tagline: z.string().nullable(),
  overview: z.string(),
  genres: z.array(z.string()),                    // names (D-11)
  posterPath: tmdbPath.nullable(),
  backdropPath: tmdbPath.nullable(),
  director: z.array(z.string()),
  cast: z.array(CastMember).max(12),
  trailerKey: z.string().nullable(),              // YouTube key (DATA-03)
  similarTmdbIds: z.array(z.number().int()),
  popularity: z.number(),
  ratings: z.object({
    imdb: z.number().min(0).max(10).nullable(),
    imdbVotes: z.number().int().nullable(),
    tmdb: z.number().min(0).max(10).nullable(),
    tmdbVotes: z.number().int().nullable(),
    rottenTomatoes: z.number().int().min(0).max(100).nullable(),
    metacritic: z.number().int().min(0).max(100).nullable(),
  }),
  disc: z.object({
    format: z.enum(["DVD", "Blu-ray", "4K UHD"]).default("DVD"),
    edition: z.string().nullable(),
    region: z.string().nullable(),
    condition: z.enum(["new", "like-new", "very-good", "good", "acceptable"]).nullable(),
    quantity: z.number().int().positive().default(1),
    notes: z.string().nullable(),
  }),
  sale: z.object({
    status: z.enum(["available", "reserved", "sold"]).default("available"),
    priceCents: z.number().int().nonnegative().nullable(),
    currency: z.string().length(3).default("CAD"),
    soldAt: isoDate.nullable(),
  }),
  ingest: z.object({
    sourcePhoto: z.string().nullable(),
    ocrTitle: z.string().nullable(),
    confidence: z.number().min(0).max(1),
    confirmedBy: z.enum(["auto", "seller", "seed"]),
    addedAt: isoDate,
    enrichedAt: isoDate,
  }),
});

export const ATTRIBUTION =
  "This product uses the TMDB API but is not endorsed or certified by TMDB.";

export const Catalog = z.object({
  $schema: z.string().optional(),
  version: z.literal(1),
  updatedAt: isoDate,
  attribution: z.literal(ATTRIBUTION),
  movies: z.array(Movie).min(1),
});

export const Seller = z.object({
  name: z.string(),
  email: z.string().email(),
  location: z.string(),
  currency: z.string().length(3).default("CAD"),
  blurb: z.string(),
  terms: z.array(z.string()),
});

export type Movie = z.infer<typeof Movie>;
export type Catalog = z.infer<typeof Catalog>;
export type CatalogInput = z.input<typeof Catalog>;
export type Seller = z.infer<typeof Seller>;
```
`[VERIFIED: probe — a subset of this schema (all enum/default/nullable/literal/regex constructs) parsed, pretty-printed errors, and exported JSON Schema]`. zod 4 `.default()` short-circuits on `undefined` and makes the output type non-optional `[CITED: zod.dev/api]`.

### `scripts/lib/catalog-io.ts` — canonical writer (D-09, D-15, DATA-07)
```typescript
import { readFileSync, writeFileSync } from "node:fs";
import { Catalog, Movie, type Catalog as CatalogT } from "../../src/lib/schema";

export const CATALOG_PATH = "data/catalog.json";

function orderKeys(shape: Record<string, unknown>, obj: Record<string, unknown>) {
  const out: Record<string, unknown> = {};
  for (const k of Object.keys(shape)) if (k in obj) out[k] = obj[k];
  return out;
}

// zod 4: nested object schemas expose `.shape`; unwrap .default()/.nullable() via `.unwrap?.()` when present
function shapeOf(s: unknown): Record<string, unknown> | null {
  const d = (s as { def?: { innerType?: unknown } }).def;
  if (d && "innerType" in d) return shapeOf(d.innerType);
  const sh = (s as { shape?: Record<string, unknown> }).shape;
  return sh ?? null;
}

export function canonicalize(cat: CatalogT): string {
  const movies = [...cat.movies]
    .sort((a, b) => a.tmdbId - b.tmdbId || (a.disc.edition ?? "").localeCompare(b.disc.edition ?? ""))
    .map((m) => {
      const ordered = orderKeys(Movie.shape, m) as Record<string, unknown>;
      for (const k of ["ratings", "disc", "sale", "ingest"] as const) {
        const sub = shapeOf(Movie.shape[k]);
        if (sub) ordered[k] = orderKeys(sub, m[k] as Record<string, unknown>);
      }
      return ordered;
    });
  return JSON.stringify(orderKeys(Catalog.shape, { ...cat, movies }), null, 2) + "\n";
}

export function readCatalog(path = CATALOG_PATH): CatalogT {
  const result = Catalog.safeParse(JSON.parse(readFileSync(path, "utf8")));
  if (!result.success) throw result.error;
  return result.data;
}

export function writeCatalog(cat: CatalogT, path = CATALOG_PATH) {
  writeFileSync(path, canonicalize(Catalog.parse(cat)));
}
```
`[VERIFIED: probe — the flat version (top-level + Movie keys) was idempotent; the nested `shapeOf` unwrap for `.default()`-wrapped sub-objects is `[ASSUMED]` (A4) — nested sub-objects here are plain `z.object`, so `.shape` is directly available and the unwrap branch is defensive]`.

### `scripts/validate.ts`
```typescript
import { readFileSync, writeFileSync } from "node:fs";
import { z } from "zod";
import { Catalog, Seller } from "../src/lib/schema";
import { CATALOG_PATH, canonicalize } from "./lib/catalog-io";

const write = process.argv.includes("--write");
const raw = JSON.parse(readFileSync(CATALOG_PATH, "utf8"));
const parsed = Catalog.safeParse(raw);
if (!parsed.success) { console.error(z.prettifyError(parsed.error)); process.exit(1); }
const cat = parsed.data;

const slugs = new Set<string>(), keys = new Set<string>();
for (const m of cat.movies) {
  if (m.id !== m.slug) fail(`id !== slug for ${m.slug}`);
  if (slugs.has(m.slug)) fail(`duplicate slug ${m.slug}`); slugs.add(m.slug);
  const k = `${m.tmdbId}::${m.disc.edition ?? ""}`;
  if (keys.has(k)) fail(`duplicate (tmdbId, edition) ${k}`); keys.add(k);
}
Seller.parse(JSON.parse(readFileSync("data/seller.json", "utf8")));

const canonical = canonicalize(cat);
if (write) writeFileSync(CATALOG_PATH, canonical);
else if (canonical !== readFileSync(CATALOG_PATH, "utf8")) fail("catalog.json is not canonical — run: npm run validate -- --write");

writeFileSync("data/catalog.schema.json",
  JSON.stringify(z.toJSONSchema(Catalog, { target: "draft-2020-12", io: "input" }), null, 2) + "\n");
console.log(`OK: ${cat.movies.length} movies`);

function fail(msg: string): never { console.error(`✖ ${msg}`); process.exit(1); }
```
`[VERIFIED: probe — check mode failed on non-canonical input; `--write` fixed it; schema file emitted with `$schema: https://json-schema.org/draft/2020-12/schema`]`.

### `scripts/seed.ts` — TMDB calls (D-13..D-15, DATA-03)
Auth: `Authorization: Bearer <v4 read access token>` on v3 endpoints `[CITED: developer.themoviedb.org/docs/authentication-application]`. Run as `npm run seed` (`tsx --env-file-if-exists=.env.local scripts/seed.ts`); env name is `TMDB_API_TOKEN` (already present locally per phase context).
```typescript
const TOKEN = process.env.TMDB_API_TOKEN;
if (!TOKEN) { console.error("TMDB_API_TOKEN missing (put it in .env.local)"); process.exit(1); }
const H = { Authorization: `Bearer ${TOKEN}`, accept: "application/json" };
const api = async <T>(path: string) => {
  const r = await fetch(`https://api.themoviedb.org/3${path}`, { headers: H });
  if (!r.ok) throw new Error(`${r.status} ${path}`);
  return (await r.json()) as T;
};

// 1. identify: /search/movie?query=&primary_release_year=   [CITED: reference/search-movie]
//    result item: id, title, original_title, release_date, popularity, vote_average, vote_count, poster_path, genre_ids
// 2. enrich in one call:                                   [CITED: reference/movie-details, docs/append-to-response]
const m = await api<TmdbMovie>(`/movie/${id}?append_to_response=credits,external_ids,videos,release_dates,similar&language=en-US`);
// fields: imdb_id, title, original_title, release_date, runtime, tagline, overview, genres[{id,name}],
//         poster_path, backdrop_path, popularity, vote_average, vote_count,
//         credits.cast[{id,name,character,profile_path,order}] (ordered by `order`), credits.crew[{name,job,department}]
//         videos.results[{key,site,type,official,name}], release_dates.results[{iso_3166_1, release_dates[{certification,type,release_date}]}],
//         similar.results[{id,…}], external_ids.{imdb_id,…}
const director = m.credits.crew.filter((c) => c.job === "Director").map((c) => c.name);
const cast = m.credits.cast.slice(0, 12).map((c) => ({ tmdbId: c.id, name: c.name, character: c.character ?? null, profilePath: c.profile_path ?? null, order: c.order }));
const trailer = m.videos.results.find((v) => v.site === "YouTube" && v.type === "Trailer" && v.official) ?? m.videos.results.find((v) => v.site === "YouTube" && v.type === "Trailer");
const us = m.release_dates.results.find((r) => r.iso_3166_1 === "US");
const certification = us?.release_dates.find((d) => d.type === 3 && d.certification)?.certification   // 3 = Theatrical
  ?? us?.release_dates.find((d) => d.certification)?.certification ?? null;                            // [CITED: reference/movie-release-dates]
const year = m.release_date ? Number(m.release_date.slice(0, 4)) : null;
const slug = `${m.id}-${kebab(m.title)}${edition ? `--${kebab(edition)}` : ""}`;
```
`kebab()`: `s.normalize("NFD").replace(/\p{M}/gu, "").toLowerCase().replace(/&/g, " and ").replace(/[^a-z0-9]+/g, "-").replace(/^-+|-+$/g, "")`. Genre names come from `m.genres[].name` (no `/genre/movie/list` call needed for details; keep the endpoint in mind for Phase 3's search results which only carry `genre_ids`). Image paths are stored verbatim (`/abc.jpg`). Response field names above are `[CITED]` from TMDB's OpenAPI reference pages; the probe could not call the API (secret-read guard), so a 5-minute smoke run of `npm run seed` against one id is the first thing the seed task should do.

### `src/lib/images.ts` (D-07) and `Poster` fallback
```typescript
const TMDB_IMG = "https://image.tmdb.org/t/p";                 // [CITED: developer.themoviedb.org/docs/image-basics]
const MODE = process.env.NEXT_PUBLIC_IMAGE_MODE === "local" ? "local" : "remote";
export type PosterSize = "w185" | "w342" | "w500" | "w780";
export function posterUrl(path: string, size: PosterSize = "w342"): string {
  return MODE === "local" ? withBasePath(`/img/tmdb/${size}${path}`) : `${TMDB_IMG}/${size}${path}`;
}
```
`Poster.tsx`: `"use client"`; `const [broken, setBroken] = useState(false)`; render `<PosterPlaceholder/>` when `!posterPath || broken`; else `<img src={posterUrl(posterPath)} alt="" width={342} height={513} loading="lazy" decoding="async" onError={() => setBroken(true)} className="h-full w-full object-cover" />`.

### `src/lib/filters.ts` — pure, unit-tested
```typescript
export const normalize = (s: string) => s.normalize("NFD").replace(/\p{M}/gu, "").toLowerCase();

export type Sort = "popularity" | "rating" | "year" | "title";
export type Avail = "available" | "sold" | "all";
export interface FilterState { q: string; genres: string[]; decades: number[]; min: number | null; avail: Avail; sort: Sort }
export const DEFAULT: FilterState = { q: "", genres: [], decades: [], min: null, avail: "available", sort: "popularity" };

export function parseParams(sp: URLSearchParams): FilterState { /* q, genre (csv kebab), decade (csv int), min, avail, sort — invalid → default */ }
export function serializeParams(s: FilterState): string {
  const p = new URLSearchParams();
  if (s.q) p.set("q", s.q);
  if (s.genres.length) p.set("genre", s.genres.map(kebab).join(","));
  if (s.decades.length) p.set("decade", s.decades.join(","));
  if (s.min != null) p.set("min", String(s.min));
  if (s.avail !== "available") p.set("avail", s.avail);
  if (s.sort !== "popularity") p.set("sort", s.sort);
  return p.toString();
}
export const isDefault = (s: FilterState) => serializeParams(s) === "";

export function applyFilters(cards: Card[], s: FilterState): Card[] {
  const q = normalize(s.q);
  return cards.filter((c) =>
    (s.avail === "all" || (s.avail === "sold" ? c.status === "sold" : c.status !== "sold")) &&
    (!q || normalize(c.title).includes(q) || (c.originalTitle && normalize(c.originalTitle).includes(q))) &&
    (!s.genres.length || c.genres.some((g) => s.genres.includes(kebab(g)))) &&
    (!s.decades.length || (c.year != null && s.decades.includes(Math.floor(c.year / 10) * 10))) &&
    (s.min == null || (c.rating != null && c.rating >= s.min)));
}

export function sortCards(cards: Card[], sort: Sort): Card[] {
  const byTitle = (a: Card, b: Card) => a.title.localeCompare(b.title, "en");
  const cmp: Record<Sort, (a: Card, b: Card) => number> = {
    popularity: (a, b) => b.popularity - a.popularity || byTitle(a, b),
    rating: (a, b) => (b.rating ?? -1) - (a.rating ?? -1) || byTitle(a, b),   // nulls last
    year: (a, b) => (b.year ?? -1) - (a.year ?? -1) || byTitle(a, b),
    title: byTitle,
  };
  return [...cards].sort(cmp[sort]);   // Array.prototype.sort is stable (ES2019); tiebreak makes it deterministic anyway
}
```
Compute genre/decade chip options from `cards` at build (`new Set`), alphabetical / ascending. Wrap `applyFilters`+`sortCards` in `useMemo([cards, state])` in the client component.

### `vitest.config.ts`
```typescript
import { defineConfig } from "vitest/config";
export default defineConfig({
  resolve: { tsconfigPaths: true },
  test: { environment: "node", include: ["src/**/*.test.ts", "scripts/**/*.test.ts"] },
});
```
`[VERIFIED: probe — 1 test using `@/lib/filters` passed in 100 ms under Vitest 5.0.3]`. Vitest 5 defaults `clearMocks: true` `[CITED: migration guide]` — irrelevant for pure-function tests.

### Local nested-prefix serve (success criterion 4)
```bash
NEXT_PUBLIC_BASE_PATH=/sale/dvds npm run build
ROOT=$(mktemp -d) && mkdir -p "$ROOT/sale" && ln -s "$PWD/out" "$ROOT/sale/dvds"
(cd "$ROOT" && python3 -m http.server 8765) &
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8765/sale/dvds/          # 200
grep -o '\(src\|href\)="/[^"]*"' out/index.html | grep -v '"/sale/dvds' || echo "no unprefixed URLs"
# every emitted asset:
for a in $(grep -o '\(src\|href\)="/sale/dvds/_next[^"]*"' out/index.html | sed 's/.*="\([^"]*\)"/\1/' | sort -u); do curl -s -o /dev/null -w "$a %{http_code}\n" "http://127.0.0.1:8765$a"; done
```
`[VERIFIED: probe — `/sale/dvds/`, `/index.html`, `/tmdb.svg`, `/.nojekyll`, and all `_next` chunks/CSS/woff2 → 200; zero unprefixed URLs]`. Python 3.14.7 is installed locally. `npx serve` is an alternative but needs the same parent-dir symlink trick.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `next lint` + `eslint` key in `next.config` | `eslint .` with flat `eslint.config.mjs`; `next build` never lints | Next 16.0 | Add `lint` script; nothing runs lint in CI unless you call it |
| `vite-tsconfig-paths` plugin | `resolve.tsconfigPaths: true` | Vite 8 (bundled by Vitest 5) | One fewer dev dep |
| `dotenv` | `node --env-file[-if-exists]` (Node 20.6 / 22.9) | — | `tsx` forwards the flag |
| `jsx: "preserve"` in tsconfig | `jsx: "react-jsx"` forced by `next build` | Next 16 | Commit upfront |
| zod 3 `zodToJsonSchema` third-party | `z.toJSONSchema` built in | zod 4 | No extra dep for D-12 |
| `lucide-react@0.x` | `1.x` — `aria-hidden` default, ESM/CJS only, brand icons removed | 2026-03-23 | Label icon-only buttons on the `<button>` |
| `closedby="any"` light dismiss | Manual backdrop-click handler | Safari lacks `closedby` (≤27.2) | ~8 lines of code |

**Deprecated/outdated:** `experimental.missingSuspenseWithCSRBailout` (14.x only — cannot disable the Suspense check in 16) `[CITED: nextjs.org/docs/messages/missing-suspense-with-csr-bailout]`; `actions/configure-pages` with `static_site_generator: next` (ARCHITECTURE anti-pattern 6, Phase 2 concern).

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | App Router `useRouter().replace()` re-applies `basePath` to a `usePathname()`-relative href exactly like `next/link` | Pattern 3 | Filter changes under `/sale/dvds/` would navigate to `/?q=…` (wrong origin path). Detect in the nested-serve UAT by typing in the search box; fallback: `router.replace(`${window.location.pathname}?${qs}`)` is basePath-agnostic |
| A2 | `@theme inline` keys named identically to the `:root` variables (`--color-bg: var(--color-bg)`) resolve correctly at runtime | Pattern 5 | Utilities like `bg-bg` could resolve to nothing; the sibling idiom `bg-[var(--color-bg)]` is unaffected. Verify visually at first `next dev`; rename theme keys if needed |
| A3 | iOS Safari still scrolls the page behind an open modal `<dialog>` (needs `overflow:hidden` on `<html>`) | Pattern 6 | Only a UX nit; the lock is harmless if unnecessary |
| A4 | zod 4 wrapped schemas (`.default()`, `.nullable()`) expose the inner schema at `def.innerType` for the recursive key-ordering helper | catalog-io | Nested key order could fall back to insertion order; still deterministic across runs because `Catalog.parse` output is built from the same code path — diff-stability holds either way. Sub-objects in the schema are plain `z.object`, so the branch is not exercised in Phase 1 |
| A5 | TMDB `release_dates.results[].release_dates[].type === 3` (Theatrical) carries the MPAA certification for US titles; some titles only have it on type 4/5 | seed.ts | Wrong/empty `certification` on a few seeds; the code falls back to any US entry with a non-empty certification. Phase 1 UI does not display certification |
| A6 | The Claude Code secret-read hook does not inspect `npm run seed`'s child command (`tsx --env-file-if-exists=.env.local …`) | Pitfall 9 | Executor cannot run the seed; D-14 already mandates a human checkpoint — Patrick runs `npm run seed` himself |

## Open Questions (RESOLVED)

1. **Does `npm run seed` survive the secret-read hook?** — RESOLVED
   - What we know: direct `node --env-file=.env.local` and `grep .env.local` are blocked for the agent.
   - What's unclear: whether a `package.json` script containing the path is also blocked.
   - Recommendation: planner keeps the seed task as `checkpoint:human-action` (D-14) with the exact command for Patrick; the executor verifies the resulting `data/catalog.json` with `npm run validate`.
   - RESOLVED: plan 01-01 Task 4 (seed) step 12 — the executor runs `npm run seed` (the `.env.local` path lives only inside the package.json script, and the token was verified present by the orchestrator on 2026-10-07); if and only if the hook still blocks it, the task raises a dynamic blocking `checkpoint:human-action` with the exact command `npm run seed && npm run validate` for Patrick and continues from the resulting `data/catalog.json` once validate reports `OK: 12 movies`.

2. **Should `eslint-config-prettier` be added now?** — RESOLVED
   - What we know: `eslint-config-next` includes no stylistic rules that conflict with Prettier defaults in the probe (0 errors, 1 semantic warning).
   - Recommendation: skip; add only if `npm run lint` and `npm run format:check` disagree.
   - RESOLVED: plan 01-02 Task 1 — not added; `eslint.config.mjs` is `eslint-config-next` core-web-vitals + typescript with the single `no-img-element` override, and the phase-end repo-wide `npm run lint && npm run format:check` pass (01-02 `<verification>`) is the trigger for revisiting.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | everything | ✓ | v22.22.1 (≥ 22.12 for Vitest 5; ≥ 20.9 for Next 16) | — |
| npm | install, scripts | ✓ | 10.9.4 | — |
| Python 3 | nested-prefix serve recipe | ✓ | 3.14.7 | `npx serve@14.2.6` |
| `gh` | not needed in Phase 1 (repo exists) | ✓ | 2.100.0 | — |
| Network to npm + fonts.googleapis (build time only) | `npm install`, `next/font/google` | ✓ (probe installed 387 pkgs, fonts downloaded) | — | — |
| `TMDB_API_TOKEN` in `.env.local` | `scripts/seed.ts` | ✓ per phase context (HTTP 200 verified 2026-10-07 by orchestrator) | v4 read token | None — seed blocked without it (D-14 checkpoint) |
| `.gitignore` coverage (`.env*`, `.cache/`, `photos/`, image globs, `out/`, `.next/`) | DATA-08 | ✓ already committed (commit 2fba8e5) | — | — |
| Sibling repo at `~/github.com/patrickclery/patrickclery.github.io` | PATTERNS.md copy sources | ✓ | — | — |

**Missing dependencies with no fallback:** none.
**Missing dependencies with fallback:** none.

## Security Domain

`security_enforcement: true`, ASVS level 1. This phase ships a static site with no auth, sessions, or server; the attack surface is the build pipeline and the client bundle.

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | — (no users) |
| V3 Session Management | no | — |
| V4 Access Control | no | — |
| V5 Input Validation | yes | zod `Catalog`/`Seller` schemas at build; `parseParams` whitelists URL keys/values (unknown `sort`/`avail` → default; `min` parsed as number in {6,7,8}) |
| V6 Cryptography | no | — (no secrets at runtime) |
| V8 Data Protection | yes | Token only in `.env.local` (gitignored) and `process.env` of `scripts/seed.ts`; never `NEXT_PUBLIC_*`; `scripts/` never imported from `src/` |
| V14 Configuration | yes | Exact dependency pins + committed lockfile; no `postinstall` scripts in the dep set; `prebuild` validate gate |

### Known Threat Patterns for this stack

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| API token leaks into `out/` bundle | Information disclosure | Read token only in Node scripts; add `grep -r "TMDB_API_TOKEN\|eyJ" out/` to the build verification (probe grep → clean); Phase 2 CI makes it permanent (DEP-03) |
| `router.replace` with attacker-controlled href (XSS via `javascript:` URLs) | Tampering | Only ever call `router.replace(pathname + "?" + URLSearchParams)`; never interpolate a raw query value into the path `[CITED: use-router docs "Good to know"]` |
| Malicious catalog entry breaks rendering (e.g. HTML in `title`) | Tampering | React escapes text; no `dangerouslySetInnerHTML`; `overview` rendered as text only |
| Poster `posterPath` pointing off-CDN | Spoofing | zod regex `^\/[\w-]+\.(jpg\|png)$` guarantees the URL stays under `image.tmdb.org/t/p/` |
| Dependency confusion / slopsquat | Tampering | Legitimacy audit above; no new or low-download packages |
| EXIF/GPS in committed photos | Information disclosure | `.gitignore` image globs outside `public/` (already committed); no photos in Phase 1 |

## Sources

### Primary (HIGH confidence — exercised in this session)
- Throwaway scaffold `scratchpad/probe` (Next 16.3.5, React 19.3.0, TS 5.9.3, Tailwind 4.3.3, zod 4.6.5, Vitest 5.0.3, lucide-react 1.52.0, tsx 4.23.15, ESLint 9.39.5/10.12.0, eslint-config-next 16.3.5/16.4.0): bare + `/sale/dvds` builds, nested-prefix HTTP serve, no-Suspense failure, bad-catalog failure, canonical-writer idempotency, JSON Schema export, Vitest alias resolution, ESLint matrix, tsconfig mutation, `Intl`/`normalize`/`--env-file` runtime checks.
- npm registry via `npm view` (versions, dist-tags, publish times, peers, engines, postinstall) — 2026-10-07.
- `gsd-tools query package-legitimacy check` — 2026-10-07.
- Sibling repo files read directly: `package.json`, `next.config.ts`, `tsconfig.json`, `postcss.config.mjs`, `src/app/layout.tsx`, `.github/workflows/deploy.yml`.
- This repo: `.gitignore` (read), `.env.local` (name only, per phase context; contents never read).

### Secondary (official docs fetched this session — `[CITED]`; seam rates `webfetch` LOW, upgraded to MEDIUM where the probe agrees)
- https://nextjs.org/docs/app/guides/static-exports (v16.4.0, updated 2026-08-09) — supported/unsupported features, `out/` layout, GitHub Pages template.
- https://nextjs.org/docs/app/api-reference/config/next-config-js/basePath — inlined at build; Link/router auto-prefix; images do not.
- https://nextjs.org/docs/app/api-reference/config/next-config-js/assetPrefix — not recommended for sub-paths; `public/` not prefixed.
- https://nextjs.org/docs/app/api-reference/functions/use-search-params — Suspense requirement; dev vs build behaviour.
- https://nextjs.org/docs/messages/missing-suspense-with-csr-bailout — fixes; disable flag is 14.x only.
- https://nextjs.org/docs/app/api-reference/functions/use-router — `replace(href, { scroll })`; XSS note.
- https://nextjs.org/docs/app/api-reference/config/eslint (updated 2026-10-05) — flat config, `next lint` removed in 16, ESLint 10 caveat.
- https://nextjs.org/docs/app/api-reference/components/font — self-hosting, `variable`, `Space_Grotesk` naming, Tailwind `@theme inline`.
- https://nextjs.org/docs/app/api-reference/file-conventions/public-folder — `public/` served from `/`.
- https://zod.dev/json-schema and https://zod.dev/api — `z.toJSONSchema` options/defaults; `.shape`, `.default()`, `z.prettifyError`.
- https://vite.dev/config/shared-options — `resolve.tsconfigPaths`.
- https://vitest.dev/guide/migration — Vitest 5 requirements and breaking changes.
- https://nodejs.org/docs/latest-v22.x/api/cli.html — `--env-file` (20.6), `--env-file-if-exists` (22.9), precedence.
- https://tailwindcss.com/docs/theme — `@theme` vs `@theme inline`, namespaces.
- https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement/showModal and …/Elements/dialog — modal semantics, `closedby`, events.
- https://caniuse.com/dialog (97.1%, Safari 15.4) and https://caniuse.com/mdn-html_elements_dialog_closedby (73%, no Safari).
- https://lucide.dev/guide/advanced/accessibility; https://github.com/lucide-icons/lucide/releases/tag/1.0.1 — v1 defaults/breaking changes.
- TMDB: https://developer.themoviedb.org/docs/authentication-application, /docs/append-to-response, /docs/image-basics, /reference/search-movie, /reference/movie-details, /reference/movie-credits, /reference/movie-videos, /reference/movie-release-dates; https://www.themoviedb.org/about/logos-attribution.

### Tertiary (LOW confidence)
- none

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — exact pins installed, built, tested, linted in the probe; registry metadata read directly.
- Architecture: HIGH — export/basePath/Suspense/JSON-import/fonts/`.nojekyll` all observed in `out/`; nested-prefix serving returned 200 for every asset.
- Pitfalls: HIGH for ESLint 10 crash, tsconfig mutation, `Intl` rounding, non-canonical-file failure (all reproduced); MEDIUM for `<dialog>` Safari behaviour (docs + caniuse only).
- TMDB seed mapping: MEDIUM — field names from official OpenAPI reference pages; live call not possible this session (secret-read guard).

**Research date:** 2026-10-07
**Valid until:** 2026-11-06 for the stack (Next 16.4.x will age into "safe" within two weeks — re-check before Phase 2); TMDB endpoint shapes are stable for months.
