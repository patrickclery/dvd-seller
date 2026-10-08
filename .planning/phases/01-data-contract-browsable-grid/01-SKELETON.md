# Walking Skeleton — DVD Seller

**Phase:** 1
**Generated:** 2026-10-07

## Capability Proven End-to-End

A buyer opens the statically exported site under any base path (`/`, `/dvd-seller`, `/sale/dvds`) and sees a poster grid of 12 real seeded DVDs loaded from `data/catalog.json`; typing in the search box filters the grid and mirrors the text into `?q=`, and reloading that URL reproduces the view.

## Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Framework | Next.js 16.3.5 App Router, `output: "export"`, `trailingSlash: true`, `images.unoptimized: true` (D-28, D-33) | Matches the sibling `patrickclery.github.io` so the two deploy the same way and can be composed; `generateStaticParams` (Phase 2) gives per-movie HTML on GitHub Pages with no rewrite rules |
| Base path | `basePath`/`assetPrefix` from `NEXT_PUBLIC_BASE_PATH` (inlined at build); `withBasePath()` in `src/lib/base-path.ts` for every non-`next/link` URL (D-28, DEP-02) | One build, three mount points (own Pages site, `/sale/dvds` compose, bare); never hard-code a path |
| Data layer | One committed `data/catalog.json` + `data/seller.json`, validated by the zod contract in `src/lib/schema.ts`; imported at build time (never fetched) and projected to a slim `Card[]` for the client (D-01..D-12, D-29, D-31) | Zero backend; git history is the audit log; the same schema is the contract for Phase 3 ingestion scripts and Phase 4 skill |
| Writer | `scripts/lib/catalog-io.ts` `writeCatalog()` — canonical form (tmdbId sort, schema key order, 2-space, LF, trailing newline), atomic tmp+rename (D-09, D-15, DATA-07) | A no-op rewrite is an empty diff; seed today and `enrich.ts` tomorrow share one writer |
| Build gate | `prebuild` → `tsx scripts/validate.ts` (schema, `id===slug`, slug uniqueness, `(tmdbId, edition)` uniqueness, canonical bytes, seller.json; emits `data/catalog.schema.json`) (D-06, D-12, D-34) | A malformed entry can never ship; JSON Schema lets non-TS tooling validate |
| Auth | None — public static catalog; the only secret (`TMDB_API_TOKEN`) lives in `.env.local` and is read solely by `scripts/seed.ts` (D-14, DATA-08) | No users, no sessions; API keys never reach the bundle |
| Client state | URL search params are the store (`q`, `genre`, `decade`, `min`, `avail`, `sort`), hand-rolled `URLSearchParams`, `router.replace(..., { scroll: false })`, consumer inside `<Suspense>` in the server `page.tsx` (D-26, D-27) | Shareable/back-navigable views for free; Suspense is mandatory for `next build` |
| Images | TMDB CDN fragments in JSON; `src/lib/images.ts` is the only URL builder (`w342` grid, `w500` detail later); `NEXT_PUBLIC_IMAGE_MODE=local` switch reserved (D-07) | Zero hosted images; CDN caches a year; switching modes never touches data |
| Styling | Tailwind v4 CSS-first; sibling `:root` tokens verbatim + `@theme inline`; Archivo (headings) + Space Grotesk (body) via `next/font/google` (self-hosted at build); dark only; `lucide-react` icons (D-16) | One visual property with the main site when mounted; no Google requests at runtime |
| Deployment target (Phase 1) | Local nested-prefix check: `NEXT_PUBLIC_BASE_PATH=/sale/dvds npm run build` served from `<tmp>/sale/dvds` by `python3 -m http.server` (`scripts/check-base-path.sh`) | Proves the mount topology before any Actions workflow exists (Phase 2 copies the sibling `deploy.yml`) |
| Tooling | TypeScript 5.9 (not 7), Vitest 5 (`resolve.tsconfigPaths`), ESLint 9.39.5 flat config + `eslint-config-next`, Prettier 3; `tsx` for scripts; Node 22 (`.nvmrc`, `engines`) (D-33, D-35) | Verified combination in RESEARCH; ESLint 10 crashes the React plugin |
| Directory layout | `src/app/*` (routes), `src/components/*` (PascalCase `export function`), `src/lib/*` (pure + build-time), `scripts/` + `scripts/lib/` (Node-only, keys allowed), `data/` (the contract), `public/` (`.nojekyll`, `tmdb.svg`) | `src/` is bundled for the browser and must never import `scripts/`; `data/` outside `src/` makes the one-way street obvious |

## Stack Touched in Phase 1

- [x] Project scaffold (Next 16 export config, TypeScript, Tailwind v4, Vitest, ESLint/Prettier, `.nvmrc`)
- [x] Routing — `/` (server page with Suspense boundary); card links already target `/movie/{slug}/` for Phase 2
- [x] Data — one real read (`data/catalog.json` → `Catalog.parse` → `getCards()`) AND one real write (`scripts/seed.ts` → `writeCatalog()` → canonical file)
- [x] UI — search input wired to URL state and the grid (plan 01-01); chips/sort/sheet (plan 01-03); card sale states (plan 01-04)
- [x] Deployment — documented local full-stack run: `npm run build && npm run check:basepath` (nested `/sale/dvds` serve returning 200 for every asset)

## Out of Scope (Deferred to Later Slices)

- Per-movie detail pages, OG meta, mailto CTA, 404 page (Phase 2 — DTL-*, SALE-01..03)
- GitHub Actions deploy, matrix builds, link checker, leaked-key CI grep (Phase 2 — DEP-01/03/04/06)
- Throttled/cached TMDB + OMDb clients, search/enrich/mark-sold scripts, golden remake set (Phase 3 — ING-*)
- The Claude Code ingestion skill and the real catalog fill (Phase 4 — SKILL-*)
- `/sale/dvds` compose job via `repository_dispatch` (Phase 5 — DEP-05)
- Local image cache implementation (v2 IMG-01; switch reserved), Fuse.js fuzzy search, actor/director search, light theme (CONTEXT.md Deferred Ideas)

## Subsequent Slice Plan

Each later phase adds one vertical slice on top of this skeleton without altering its architectural decisions:

- Phase 1 / plan 01-02: contract hardening — tests for schema, writer determinism, and the validate gate; ESLint/Prettier wired
- Phase 1 / plan 01-03: filters, sort, chips, mobile sheet, result count, empty state — all on the existing URL-state seam
- Phase 1 / plan 01-04: card rating badge, price/condition chips, SOLD/Reserved treatments, placeholder poster — on the existing `Card` projection
- Phase 2: `/movie/[slug]/` via `generateStaticParams` over `getAllMovies()`; `withBasePath()` for OG images; sibling `deploy.yml` with `NEXT_PUBLIC_BASE_PATH`
- Phase 3: `scripts/{search,enrich,mark-sold}.ts` writing through `writeCatalog()` and validated by the same `Catalog` schema
- Phase 4: `.claude/skills/ingest-dvds/` sequencing the Phase 3 scripts
- Phase 5: main-site compose job building this repo with `NEXT_PUBLIC_BASE_PATH=/sale/dvds`
