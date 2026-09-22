# Phase 1: Data Contract & Browsable Grid - Context

**Gathered:** 2026-09-21
**Status:** Ready for planning
**Mode:** `--auto` (advisor mode active; all gray areas auto-selected, recommended options chosen without prompting)

<domain>
## Phase Boundary

Deliver the shared data contract and the browsable grid on top of it: a zod schema in `src/lib/schema.ts` that both the SPA and the future ingestion scripts import, a seeded `data/catalog.json` (5–10 real titles, at least one sold), a `validate` prebuild step that fails the build on bad data, and a phone-first Next.js static-export poster grid with genre/decade/rating/availability filters, sort, instant title search, URL-persisted state, result count, clear-all, empty state, SOLD treatment, price/condition chips, header counts + seller blurb — all working under a configurable `NEXT_PUBLIC_BASE_PATH`.

NOT in this phase: per-movie detail pages, mailto CTA, OG meta, deploy workflow (Phase 2); TMDB/OMDb ingestion scripts beyond a one-off seed helper (Phase 3); the skill (Phase 4); `/sale/dvds` mount (Phase 5).

</domain>

<decisions>
## Implementation Decisions

### Schema shape (DATA-01..05, DATA-07)
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

### Seed data (DATA-01, success criterion 1)
- **D-13:** Seed with ~10 real, well-known titles fetched from TMDB via a one-off `scripts/seed.ts` (plain `fetch`, `TMDB_API_TOKEN` v4 read token from `.env.local`). Include deliberate variety: at least one `sold`, one `reserved`, one with `priceCents: null` ("Ask"), one with no poster (placeholder path), one Blu-ray, titles spanning several decades and genres, one remake pair (e.g., The Thing 1982) so the year is exercised. — **Reversibility:** reversible — seed rows are replaced by real ingestion in Phase 4.
- **D-14:** **External dependency:** Patrick must create a free TMDB account and put `TMDB_API_TOKEN=<v4 read access token>` in `.env.local` before the seed task runs. The planner MUST make this an explicit checkpoint (`checkpoint:human-action`) rather than assuming the key exists. No key is ever committed or read from `NEXT_PUBLIC_*`.
- **D-15:** `scripts/seed.ts` is throwaway-grade but must write through the same zod-validated, deterministic writer that `validate --write` uses, so Phase 3's `enrich.ts` can replace it without touching the data file format. Do not build the throttled/cached TMDB client here — that is Phase 3.

### Visual design & card layout (CAT-01, CAT-02, CAT-11, SALE-04)
- **D-16:** Inherit the sibling site's look so the two feel like one property when mounted: dark slate palette (`--color-bg #0F172A`, `--color-surface #1E293B`, `--color-border #334155`, `--color-text #F8FAFC`, `--color-text-muted #94A3B8`, `--color-accent #22C55E`, `--color-accent-warm #F59E0B`), Archivo for headings, Space Grotesk for body, `lucide-react` icons, Tailwind v4 CSS-first tokens in `globals.css`. Dark is the only theme (movie-store feel; no light mode work). — **Reversibility:** reversible — tokens live in one file.
- **D-17:** Card = poster in a fixed `aspect-[2/3]` box with `object-cover`, `loading="lazy"`, `decoding="async"`; caption BELOW the poster (title, year · rating badge, price chip + condition chip). No hover-only information (mobile has no hover); hover adds a subtle border-accent like the sibling's project cards. Whole card is a link (to the Phase 2 detail route `/movie/{slug}/`; in Phase 1 the link exists and may 404 until Phase 2).
- **D-18:** Missing poster → placeholder card: surface-colored 2:3 box with the title (and year) centered in Archivo, plus a faint `Film` icon. No external placeholder service.
- **D-19:** SOLD treatment: poster `grayscale` + reduced opacity, diagonal "SOLD" ribbon in `--color-accent-warm` across the top-left corner, price chip replaced by "Sold". Reserved: small warm-colored "Reserved" pill over the poster, no desaturation.
- **D-20:** Grid columns: 2 at ≥360px, 3 at `sm`, 4 at `md`, 5 at `lg`, 6 at `xl`; gap 3–4; container `max-w-7xl`. Touch targets ≥44px on all controls.

### Filter/sort/search UI (CAT-03..CAT-10)
- **D-21:** Desktop (`md+`): a sticky top toolbar with search input (left), sort `<select>` (right), and a horizontally scrollable row of filter chips beneath (genre multi-select chips, decade chips, rating-threshold chips `Any | 6+ | 7+ | 8+`, availability segmented control `Available | Sold | All`). Mobile: same sticky toolbar with search + sort, plus a "Filters (n)" button that opens a bottom sheet containing the same chip groups and a "Show X results" apply button. One shared chip component; the sheet is just a different container.
- **D-22:** Sort options: `Popularity` (default, TMDB popularity desc), `Rating` (displayed rating desc, nulls last), `Year` (newest first), `Title` (A→Z). Sorting is stable (secondary key title).
- **D-23:** Search is client-side substring match on `title` and `originalTitle`, diacritics-stripped (`normalize("NFD")` + strip combining marks) and case-insensitive, applied on every keystroke with no debounce needed at a few hundred items. No Fuse.js in Phase 1.
- **D-24:** Result count line "Showing 42 of 318" sits above the grid with a "Clear all" link that appears only when any filter/search/sort deviates from default. Empty state: centered message "No matches" + "Clear filters" button.
- **D-25:** Default availability filter = `Available` (which includes `reserved`). Sold titles are hidden unless the visitor picks `Sold` or `All`.

### URL state (CAT-09)
- **D-26:** Hand-rolled `URLSearchParams` (no `nuqs` dependency). Keys: `q`, `genre` (comma-separated kebab names), `decade` (comma-separated, e.g. `1990,2000`), `min` (rating threshold), `avail` (`sold|all`; omitted for default), `sort` (`rating|year|title`; omitted for default). Defaults are omitted so the canonical grid URL is bare. Updates use `router.replace` (no history spam per keystroke); back-navigation from a detail page restores the view because the state is in the URL.
- **D-27:** The component that reads `useSearchParams` lives inside a `<Suspense>` boundary in the server `page.tsx` from day one (build-time failure otherwise). Filter/sort/search logic is pure functions in `src/lib/filters.ts` with unit tests; the client component stays thin.

### Base path & static export (DEP-02, CAT-13)
- **D-28:** `next.config.ts`: `output: "export"`, `trailingSlash: true`, `images: { unoptimized: true }`, `basePath` and `assetPrefix` from `process.env.NEXT_PUBLIC_BASE_PATH ?? ""`. `src/lib/base-path.ts` exports `withBasePath(path)`; every non-`next/link` URL (OG images later, manual `<a>`, any `fetch`) goes through it. TMDB CDN URLs are absolute and bypass it.
- **D-29:** Catalog is imported at build time (`import catalog from "@/../data/catalog.json"` parsed through zod in `src/lib/catalog.ts`); the index page passes a slim card projection (slug, title, year, posterPath, rating + source, genres, decade, status, priceCents, currency, condition, popularity) to the client component — never the full `Movie[]`.
- **D-30:** `public/.nojekyll` committed; CI-side checks (matrix build, link checker) are Phase 2.

### Seller configuration (CAT-12)
- **D-31:** Seller-facing config lives in `data/seller.json` validated by a `Seller` zod schema in `src/lib/schema.ts`: `{ name, email, location, currency: "CAD", blurb, terms }` (blurb = one paragraph shown in the header; terms = pickup/shipping/payment one-liners). JSON, not TS, so other sellers can edit it without touching code. `email` is used by Phase 2's mailto. Phase 1 ships Patrick's real values (Montreal, CAD).
- **D-32:** Header shows: site title ("DVDs for Sale" — final wording is Claude's discretion), "N available · M total" counts computed at build, the seller blurb, and the TMDB attribution notice + logo in the footer of the root layout (DTL-08 is Phase 2, but the attribution footer is trivial and lands with the layout now).

### Repo hygiene & tooling (DATA-06, DATA-08)
- **D-33:** Stack pinned per `.planning/research/STACK.md`: Next.js 16.x, React 19.x, TypeScript 5.9.x (not 7), Tailwind 4.x via `@tailwindcss/postcss`, zod 4.x, lucide-react, `tsx` for scripts, Node 22 (`.nvmrc` + `engines`). `npm` with committed lockfile. Vitest for `filters.ts` and schema tests.
- **D-34:** `package.json` scripts: `dev`, `build`, `prebuild` → `validate`, `validate` → `tsx scripts/validate.ts`, `seed` → `tsx scripts/seed.ts`, `test`. `.gitignore`: `.env*`, `.cache/`, `photos/`, `*.jpg|*.jpeg|*.png|*.heic` outside `public/`, `out/`, `.next/`, `node_modules/`.
- **D-35:** Conventions match the sibling repo: PascalCase `.tsx` components with `export function`, inline prop types, 2-space indent, trailing commas, `@/*` alias. Add Prettier + ESLint (next/core-web-vitals) since this repo will have scripts and tests — light config, no custom rules.

### Claude's Discretion
- Exact copy for the header title, empty-state text, and chip labels.
- Whether the bottom sheet is a hand-rolled `<dialog>` or a small headless component — no new heavy UI library.
- Skeleton/loading treatment for the Suspense fallback.
- Test runner details and file layout under `src/lib/__tests__/` vs co-located.
- Whether `scripts/validate.ts` and the deterministic writer are one file with flags or two modules.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project definition
- `.planning/PROJECT.md` — Core value, constraints (zero hosting, no scraping, non-commercial TMDB posture), Key Decisions table
- `.planning/REQUIREMENTS.md` — DATA-01..08, CAT-01..13, SALE-04, DEP-02 are this phase's requirement IDs
- `.planning/ROADMAP.md` § "Phase 1" — goal and the 5 success criteria that verification will check

### Research (already done — do not re-research these)
- `.planning/research/ARCHITECTURE.md` § "Proposed `catalog.json` Entry Schema" — the zod schema this phase adopts (with D-02/D-03/D-10 tweaks); § Patterns 1–5 (build-time import + projection, `generateStaticParams`, Suspense-wrapped URL state, env basePath + `withBasePath()`, image-mode module); § Anti-Patterns 1–3, 5, 6
- `.planning/research/STACK.md` — verified versions (Next 16.3.x, React 19.3, Tailwind 4.3, TS 5.9, zod 4.6), TMDB image CDN sizes and hotlink/attribution policy, "what not to use"
- `.planning/research/FEATURES.md` § "Table Stakes → Visitor SPA — library grid / filter / collection level" — the behaviors CAT-* encode, with UX notes (no hover-only info, fixed aspect boxes, default Available filter)
- `.planning/research/PITFALLS.md` — Pitfall 2 (five sub-path static-export failures), Pitfall 5 (duplicates/slug churn), Pitfall 6 (secrets/photos/EXIF in repo), UX pitfalls (layout shift, filters resetting on back)
- `.planning/research/SUMMARY.md` — executive summary and phase implications

### Sibling site (conventions to mirror)
- `~/github.com/patrickclery/patrickclery.github.io/next.config.ts` — `output: "export"`, `trailingSlash: true`, `images.unoptimized`
- `~/github.com/patrickclery/patrickclery.github.io/src/app/globals.css` — design tokens to copy (D-16)
- `~/github.com/patrickclery/patrickclery.github.io/src/app/layout.tsx` — font loading pattern (Archivo + Space Grotesk via `next/font/google`)
- `~/github.com/patrickclery/patrickclery.github.io/src/components/Projects.tsx` — card styling idiom (rounded-xl, surface bg, border → accent on hover, lazy `<img>`)
- `~/github.com/patrickclery/patrickclery.github.io/CLAUDE.md` — stack + conventions summary

### External docs (verify at plan time, versions move)
- Next.js static exports guide, `basePath`, `useSearchParams` Suspense requirement (URLs listed in `.planning/research/SUMMARY.md` § Sources)
- TMDB image basics (`image.tmdb.org/t/p/{size}`) and attribution requirements

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None in this repo (greenfield; only `.planning/` and `.claude/CLAUDE.md` exist).
- From the sibling repo (copy, don't import): design tokens in `globals.css`, font setup in `layout.tsx`, card idiom in `Projects.tsx`, `next.config.ts` shape.

### Established Patterns
- Sibling conventions: App Router, server components by default, `export function` PascalCase components, inline prop types, Tailwind utility classes referencing CSS variables (`bg-[var(--color-surface)]`), `lucide-react` icons, plain `<img loading="lazy">` (no `next/image`).
- No linter/formatter in the sibling; this repo adds light Prettier + ESLint (D-35) because it will carry scripts and tests.

### Integration Points
- `data/catalog.json` and `data/seller.json` are the only inputs to the SPA; `src/lib/schema.ts` is the only contract Phase 3 scripts will import.
- `src/lib/images.ts`, `src/lib/base-path.ts`, `src/lib/filters.ts`, `src/lib/catalog.ts` are the seams Phase 2 (detail pages) will reuse unchanged.
- Card link target `/movie/{slug}/` is reserved for Phase 2's `generateStaticParams` route.

</code_context>

<specifics>
## Specific Ideas

- "Basically an IMDb clone for selling DVDs" and "each movie should have a page similar to Plex" — poster-forward, dark, dense grid; borrow seerr's grid idioms (research verdict), not its code.
- Phone-first: buyers browse on the go; 2 columns at 360px is the design anchor.
- Filter down to a title in under three clicks (PROJECT.md success metric) — chips over dropdowns, defaults omitted from the URL.
- Seller is in Montreal; currency CAD.
- The site must never call TMDB at runtime; the only external requests visible in the network tab are `image.tmdb.org` image loads.

</specifics>

<deferred>
## Deferred Ideas

- Actor/director search, "more in this collection", trailer/TMDB/Letterboxd links, recently-added row, box-set entry type, wishlist mailto, density toggle, `/` shortcut — all v2 in REQUIREMENTS.md; schema fields for cast/similar/videos are stored in Phase 1 so these are additive later.
- Local image cache mode — switch reserved in `images.ts` (D-07); implementation is v2 (IMG-01).
- Fuse.js fuzzy search — only if UAT shows misspellings matter.
- Light theme — not planned; dark only.
- Emailing TMDB about the commercial grey area — optional, recorded in PROJECT.md Key Decisions; not blocking.

</deferred>

---

*Phase: 01-data-contract-browsable-grid*
*Context gathered: 2026-09-21*
