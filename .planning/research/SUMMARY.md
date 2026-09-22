# Project Research Summary

**Project:** DVD Seller
**Domain:** Static, zero-hosting movie-catalog SPA (GitHub Pages) fed by a committed JSON file, populated by a Claude Code skill that reads DVD-spine photos and enriches from TMDB
**Researched:** 2026-09-21
**Confidence:** MEDIUM-HIGH

## Executive Summary

This is a two-halves product joined by one file. The **publishing half** is a Next.js 16 static export (`output: "export"`, `trailingSlash: true`, env-driven `basePath`) that imports `data/catalog.json` at build time, prerenders one detail page per title via `generateStaticParams`, filters a few hundred cards in memory on the client, and hotlinks posters from `image.tmdb.org`. The **ingestion half** is a project-scoped Claude Code skill (`.claude/skills/dvd-ingest/`) that reads spine photos with the model's own vision, then sequences typed `tsx` scripts (search, enrich, validate, append, mark-sold) that own all network, cache, throttle and write logic. The visitor site makes zero API calls and holds zero secrets. Experts in this space (Plex, tinyMediaManager, Libib) converge on a *propose, review, commit* flow for matching; the skill must do the same.

**Prior-art verdict: BUILD.** A verified GitHub survey found no maintained, popular, static movie-collection catalog. Every project with meaningful stars (Yamtrack 3.6k, Ryot 3.6k, seerr 12.7k, Watcharr 1.5k, Movary 779, DVinyl 224) is a server-plus-database app that violates the zero-hosting constraint, and none has a for-sale/sold state or image-based ingestion. Flat-HTML "movie collection website" repos are 0-5 stars, unlicensed and abandoned. Borrow UX conventions from seerr (poster grid, backdrop hero, cast row) and physical-media fields from DVinyl; import no code.

**Key risks and mitigations.** (1) TMDB's free key and OMDb's data are non-commercial; a DVD-for-sale catalog is a grey area. Posture: no ads, no checkout, no affiliate links, exact TMDB attribution notice plus logo in the root layout, `fetchedAt` on every entry for the 6-month cache rule; record this decision in PROJECT.md. (2) TMDB text search returns the popular remake first (*The Thing*, *Halloween*, *Dune*); the skill must search with `primary_release_year` when a year is known, detect title collisions, and route anything ambiguous to a batched seller confirmation instead of ever taking `results[0]`. (3) Static export under a sub-path breaks in five independent ways (basePath not applied to `<img>`, `useSearchParams` without Suspense failing only in `next build`, missing `.nojekyll`, case-sensitive slugs, dynamic routes without `generateStaticParams`); a CI matrix build with a link checker catches all five. (4) The `/sale/dvds` mount from a separate repo has cross-repo permission and staleness traps; ship the repo's own Pages site first, compose into the main site last.

## Key Findings

### Prior Art (user-requested check)

**Verdict: BUILD, borrowing UX only.** Method: `gh search repos` plus `gh api repos/...` verification on 2026-09-21 across movie/DVD/Blu-ray collection, Letterboxd-clone and TMDB topics. Confidence HIGH that no popular static option exists; MEDIUM that no niche repo was missed.

| Candidate | Stars | Why it does not fit |
|---|---|---|
| seerr (Overseerr successor) | 12,652 | Next.js + Node request manager for Plex/Jellyfin; server + DB. Steal its grid/hero/cast UX idioms (MIT, Tailwind) |
| Yamtrack | 3,618 | Django media tracker; server + DB; no sale state |
| Ryot | 3,604 | Rust + Postgres tracker; server + DB |
| Watcharr | 1,508 | Go + SQLite watched list; server |
| flox | 1,351 | Laravel + Vue; archived |
| MediaTracker | 934 | Node + SQLite; stale ~19 months |
| Movary | 779 | PHP + MySQL watch tracker; server |
| DVinyl | 224 | Closest conceptual match (physical DVD/Blu-ray/vinyl, TMDB plugin, barcode scan) but Node + DB + Docker; borrow its format/edition/condition fields |
| php4dvd | 86 | Only repo with DVD own/loaned semantics; PHP + MySQL, scrapes IMDb (ToS risk) |
| Static attempts (jsonmc, BluRayCollection, etc.) | 0-5 | Unlicensed, abandoned |

No Hugo/Astro/Eleventy movie-collection theme above 10 stars exists. Nothing anywhere does photo/spine ingestion. The visitor half is roughly two pages; adapting a server app down to static is more work than writing it against the sibling site's conventions.

### Recommended Stack

Match the sibling `patrickclery.github.io` (Next.js App Router static export, React 19, Tailwind v4, lucide-react, TypeScript strict) but start greenfield on Next 16 and Node 22. TMDB is the sole source of record; OMDb is an optional single-field enrichment for the true IMDb rating; IMDb itself is link-out only. See STACK.md for verified versions and the full provider comparison.

**Core technologies:**
- Next.js 16.3.5 (`output: "export"`, `trailingSlash: true`, `basePath` from `NEXT_PUBLIC_BASE_PATH`, `images.unoptimized`): static generator with per-title prerendered deep links; same config semantics as the sibling
- React 19.3 + TypeScript ^5.9 (not 7.x; Next tooling compatibility unverified) + Tailwind 4.3 via `@tailwindcss/postcss`
- zod 4.6: **the single shared schema** in `src/lib/schema.ts` is the contract between skill and SPA; `next build` fails loudly on a malformed entry, and the skill refuses to write one
- TMDB API v3 with the v4 Bearer read token; one call per title: `/movie/{id}?append_to_response=credits,external_ids,videos,release_dates,similar`; browser builds `https://image.tmdb.org/t/p/{w342|w500|w185}{path}` from stored path fragments
- OMDb (optional, `OMDB_API_KEY`): `imdbRating` by `imdb_id`, 1,000 req/day free, stored as a snapshot with `fetchedAt`; when absent, show TMDB `vote_average` labelled as TMDB
- Ingestion scripts: `tsx`, `p-throttle`/`p-limit` (4 concurrent), plain `fetch`, on-disk `.cache/` of raw responses. No Anthropic SDK: vision happens inside Claude Code's `Read` tool
- Plain `Array.filter` + `useMemo` for search/filter; Fuse.js only if UAT shows misspellings matter
- GitHub Actions: copy the sibling `deploy.yml`, set Node 22, add `NEXT_PUBLIC_BASE_PATH` and a `prebuild` validate step

**TMDB licensing posture (must be logged in PROJECT.md):** free key is non-commercial only; TMDB staff's stated test is monetizing TMDB content (ads, paid features, $149/mo commercial tier). An unlisted personal listing with no ads, no checkout and correct attribution matches the "free public webpage" case staff have approved. Keep it that way; revisit if the skill is ever sold to other sellers or ads/payments appear. OMDb (CC BY-NC) and IMDb datasets carry the same clause.

### Expected Features

Two products share one data file; PROJECT.md's Active Requirements are all P1. See FEATURES.md for the full landscape and competitor matrix.

**Must have (table stakes):**
- Poster grid with title/year/rating/price chip and a visible SOLD state (desaturate + ribbon; default filter hides sold), placeholder poster, lazy loading, phone-first 2-column layout
- Filters: genre (multi), decade, rating threshold, availability; sorts: popularity (default), rating, year, title; instant title search; result count; clear-all; **filter state persisted in the URL** (shareable, survives back navigation)
- Static-exported detail page per title: backdrop hero, poster, badges (year/runtime/genres/certification), tagline, synopsis, director, top 8-12 cast with headshots, IMDb link-out, sale block (price/condition/format), sold banner that keeps the page live
- Contact CTA with no backend: prefilled `mailto:` (`Interested in: {Title} ({Year})` + deep link); seller contact from config
- Open Graph meta per movie page: shared links in iMessage/WhatsApp are the distribution channel for an unlisted site
- Header with counts and a seller "how to buy" blurb; TMDB attribution in the root layout footer
- Ingestion: many photos in, structured per-spine read, year/edition disambiguation, full TMDB enrichment, dedupe on `tmdbId`, confidence flags, **preview/dry-run before write**, batch defaults (condition/price/format/region), mark-sold/edit subcommand, reusable by other sellers

**Should have (competitive, v1.x):**
- Actor/director search (trivial once cast is in the JSON; decide in schema phase)
- "More in this collection" cross-sell from TMDB `similar` intersected with the catalog
- Trailer / TMDB / Letterboxd link-outs (one-liners once `videos` is stored)
- Format/region/edition badges; box-set entry type via `belongs_to_collection`
- Recently-added row; multi-title wishlist `mailto:` via `localStorage`; ingestion `--refresh`

**Defer (v2+):**
- Local image cache mode (required by PROJECT.md as opt-in; reserve the config flag in v1, build when there is a reason)
- Collection stats page, density toggle, `/` keyboard shortcut
- Rotate-and-reread vision fallback (only if misread rate is high in practice)
- Skill packaging as a plugin for other sellers (after Patrick's own catalog validates the flow)

**Anti-features to refuse:** cart/checkout, accounts, comments, live TMDB calls from the browser, IMDb scraping, committed posters by default, YouTube iframes, pagination, analytics scripts, fully automatic ingestion with no review step.

### Architecture Approach

Two loosely coupled halves share exactly one contract, `data/catalog.json` and its zod schema. Ingestion runs offline on the seller's machine with API keys; publishing runs in CI and the visitor browser with none. `data/` sits at repo root (skill and scripts write it, SPA only reads it); `scripts/lib/` (network, cache, keys) is kept apart from `src/lib/` (bundled to the browser), sharing only the pure zod `schema.ts`. Filter state lives in URL search params behind a mandatory `<Suspense>` boundary; the grid receives a slim card projection, detail pages get full entries. `src/lib/images.ts` is the only module that knows about the CDN or the image mode; `withBasePath()` is the only way to build a non-`Link` URL. See ARCHITECTURE.md for the full schema and code patterns.

**Major components:**
1. `src/lib/schema.ts` + generated `data/catalog.schema.json`: the shared contract; metadata fields are regenerable from TMDB, `disc`/`sale`/`slug` are seller-owned and never overwritten by re-enrichment
2. `data/catalog.json`: single source of truth, sorted by `tmdbId` with stable key order so git diffs show only real changes; git history is the audit log
3. `.claude/skills/dvd-ingest/` (SKILL.md + reference + examples): vision read in-model, sequences scripts, owns the human confirmation step; keys from env, paths via `${CLAUDE_PROJECT_DIR}`
4. `scripts/`: `ingest.ts` (titles to ranked TMDB candidates, no writes), `enrich.ts` (one `append_to_response` call + optional OMDb, merge-preserving seller fields), `validate.ts` (zod + uniqueness, wired as `prebuild`), `mark-sold.ts`, opt-in `cache-images.ts`; `scripts/lib/{tmdb,omdb,cache}.ts` with `p-limit(4)`, SHA-keyed disk cache, OMDb daily-budget guard
5. `src/app/page.tsx` + `CatalogBrowser` (client): in-memory filter/sort/search mirrored to `?genre=&sort=&q=`
6. `src/app/movie/[slug]/page.tsx`: `generateStaticParams` over the catalog, `dynamicParams = false`, slug `${tmdbId}-${kebab(title)}` written once at ingest and frozen
7. `.github/workflows/deploy.yml`: sibling mirror plus `NEXT_PUBLIC_BASE_PATH`, validate, `.nojekyll` assertion

**Ingestion flow (propose, review, commit):** vision read to `{rawText, titleGuess, yearGuess, editionCues, confidence}` per spine; `ingest.ts` searches TMDB with `primary_release_year` when a year exists, else plain query with title-collision detection; auto-accept only when vision confidence >= 0.8 AND the match is unambiguous; everything else goes to one batched `AskUserQuestion` ("pick 1a The Thing (1982) / 1b The Thing (2011) / skip"); `enrich.ts` fetches and maps; dedupe on `(tmdbId, edition)`; `CatalogSchema.parse` hard-fails before any write; atomic write; git commit; seller pushes; Pages rebuilds.

**Deploy topology:** because `patrickclery.github.io` has `CNAME = patrickclery.com`, this repo's project Pages site publishes at `patrickclery.com/dvd-seller/` with zero coordination. Recommendation: **(a) own project Pages site for v1** (`NEXT_PUBLIC_BASE_PATH=/dvd-seller`), then **(b) main-site compose as the final phase**: the main repo's workflow checks out `patrickclery/dvd-seller`, builds with `NEXT_PUBLIC_BASE_PATH=/sale/dvds`, copies `out/` into `out/sale/dvds/`; this repo fires `repository_dispatch` with a fine-grained PAT. Same code, different env var. Never put a `CNAME` in this repo; never use `actions/configure-pages` with `static_site_generator: next` (it injects a hard-coded basePath).

### Critical Pitfalls

Top five from PITFALLS.md (nine documented there, with a "looks done but isn't" checklist).

1. **Non-commercial TMDB/OMDb licence vs. a for-sale catalog.** Decide and record the non-commercial posture in PROJECT.md; attribution notice + logo in root layout; `fetchedAt` on entries; no ads/checkout/affiliate; never scrape imdb.com.
2. **Remake/wrong-year matching that commits silently.** `primary_release_year` first; detect normalized-title collisions in top 10; never `results[0]`; batched confirmation queue; persist `confirmedBy: "seller"` so re-runs never re-ask; seed a golden test set of 15+ hard spines (remakes, sequels, box sets, foreign editions, glare) before writing the matcher.
3. **Static export under a sub-path breaks five ways at once.** One `NEXT_PUBLIC_BASE_PATH` + `withBasePath()` for every non-`Link` URL; build-time JSON import (no client `fetch`); every `useSearchParams` consumer inside `<Suspense>`; `.nojekyll` asserted in `out/`; lowercase slug regex test; CI matrix build (`""` and `/sale/dvds`) with a link checker.
4. **Duplicates and slug churn on re-ingest.** Primary key `tmdbId`; slug generated once and frozen in JSON; one zod schema shared by both halves; deterministic sort and key order; `enrich.ts` merges rather than replaces seller-owned fields.
5. **Secrets, spine photos and EXIF GPS in a public repo.** Zero `NEXT_PUBLIC_*` keys; CI grep of `out/` for `api_key`; `.gitignore` image globs and `.env*`; photos accepted from any path outside the repo; `gitleaks` in the workflow.

Also material: two-repo composition traps (cross-repo artifact permissions, 90-day artifact expiry, stale main-site builds, duplicate CNAME); local-image-cache repo bloat (fetch at build time into `out/`, never Git LFS, which Pages does not serve); vision reads that are fluently wrong (adjacent-spine bleed, "Alien" vs "Aliens"; keep `rawText` for audit, track acceptance rate per run); skill portability (`${CLAUDE_SKILL_DIR}`/`${CLAUDE_PROJECT_DIR}`, env-only keys, `--dry-run`, clean-temp-dir CI test).

## Implications for Roadmap

Dependencies flow from the contract outward. Suggested six phases; phases 2 and 4 can run in parallel once phase 1 exists.

### Phase 1: Schema, repo hygiene, and seed data
**Rationale:** Nothing downstream can be built or tested without a real entry to render and a schema both halves obey. Locking `sale`/`disc` fields, `tmdbId` as primary key and frozen slugs now avoids the two most expensive migrations (duplicates, slug churn). Repo hygiene must exist before the first photo is ever near the repo.
**Delivers:** `gh repo create patrickclery/dvd-seller`; `src/lib/schema.ts` (zod) + generated `data/catalog.schema.json`; `data/catalog.json` with 5-10 hand-enriched entries including at least one `sold`; `scripts/validate.ts` wired as `prebuild`; `.gitignore` (images outside `public/`, `.env*`, `.cache/`, `photos/`); `.nojekyll`; PROJECT.md decision entries for TMDB non-commercial posture and deploy topology (a) then (b).
**Addresses:** Catalog JSON schema; sale state; `fetchedAt`/`source` provenance; cast stored in JSON (enables actor search later).
**Avoids:** Pitfalls 1 (fields), 5 (identity/slug/schema), 6 (hygiene), 7 (cache-mode design decision: build-time fetch, never commit).

### Phase 2: Static SPA skeleton with grid and detail pages
**Rationale:** The visitor product is the thing being validated; with seed data it can be built end-to-end and verified under a sub-path before any ingestion complexity. Must come before deploy so there is something to ship.
**Delivers:** `next.config.ts` (export, trailingSlash, env basePath, unoptimized images); `lib/catalog.ts`, `lib/images.ts`, `lib/base-path.ts`, `lib/filters.ts` (pure, unit-tested); Suspense-wrapped `CatalogBrowser` with URL-mirrored filters/sorts/search, result count, clear-all, sold overlay, price chip, placeholder poster, lazy images, phone-first grid; `movie/[slug]` with `generateStaticParams` and `dynamicParams = false`; Plex-style detail (hero, badges, synopsis, director, cast, IMDb link, sale block, sticky mailto CTA, sold banner); Open Graph meta; `not-found.tsx`; TMDB attribution footer in root layout. Verify `NEXT_PUBLIC_BASE_PATH=/sale/dvds npm run build` served from a nested folder locally.
**Uses:** Next 16, React 19, Tailwind 4, lucide-react, zod parse at build.
**Implements:** Patterns 1-5 from ARCHITECTURE.md.
**Avoids:** Pitfall 2 (all five sub-path failures), UX pitfalls (sold state, contact path, filters on back, labelled rating source).

### Phase 3: Deploy as own project Pages site (topology a)
**Rationale:** A live URL from here on means every later phase ships continuously through the exact workflow the user already trusts, with no cross-repo secrets. Also proves the build before the compose phase depends on it.
**Delivers:** `.github/workflows/deploy.yml` mirrored from the sibling (Node 22, `npm ci`, `prebuild` validate, `NEXT_PUBLIC_BASE_PATH=/dvd-seller`, `.nojekyll` assertion, `gitleaks`, `api_key` grep of `out/`, matrix build + link check); live at `patrickclery.com/dvd-seller/`.
**Avoids:** Pitfall 2 (CI verification), Pitfall 6 (secrets scanning), the "second Pages URL left broken" debt item.

### Phase 4: Ingestion scripts (CLI-testable, no vision)
**Rationale:** Scripts are deterministic and testable from a hand-typed title list before any vision work; the golden test set for remakes must exist before the matcher is trusted. Can run in parallel with Phase 2.
**Delivers:** `scripts/lib/{cache,tmdb,omdb}.ts` (Bearer auth, `p-limit(4)`, 429 backoff, SHA-keyed disk cache, OMDb daily budget guard); `ingest.ts` (year-scoped search, collision detection, ranked candidates with scores, no writes); `enrich.ts` (single `append_to_response` call incl. credits/external_ids/videos/release_dates/similar, optional OMDb, merge that preserves `slug`/`disc`/`sale`); `mark-sold.ts`; dedupe on `(tmdbId, edition)`; atomic write; golden test set with >= 15 hard titles and a precision check that all remake pairs surface as ambiguous.
**Uses:** tsx, p-limit/p-throttle, plain fetch, zod.
**Avoids:** Pitfall 4 (remakes), Pitfall 5 (dedupe, merge), Anti-Pattern 4 (agent calling APIs via ad-hoc curl).

### Phase 5: Ingestion skill and catalog fill
**Rationale:** Depends on Phase 4. This is where the real shelf photos turn into hundreds of entries, and where the seller learns whether the confidence gate is tuned right.
**Delivers:** `.claude/skills/dvd-ingest/SKILL.md` (frontmatter with trigger phrases, `disable-model-invocation: true`, `allowed-tools` scoped to `npx tsx` scripts and catalog git ops), `reference.md` (spine-reading guide: rotated text, box sets, noise words, edition cues, confidence rubric), `examples.md`; structured per-spine vision output with verbatim `rawText`; batched `AskUserQuestion` confirmation; `--dry-run`/`--review`; batch defaults (condition/price/format/region); box sets as one `set` entry; per-run acceptance-rate report; catalog path resolution (arg, `DVD_CATALOG_PATH`, `${CLAUDE_PROJECT_DIR}/data/catalog.json`); README "install for another seller" + clean-temp-dir `--dry-run` CI job. Run on the full collection.
**Avoids:** Pitfall 8 (fluent misreads), Pitfall 9 (portability), Pitfall 6 (photos outside repo).

### Phase 6: Polish with real data, then mount at /sale/dvds (topology b)
**Rationale:** Filter tuning, cross-sell rows and box-set cards only make sense with hundreds of real entries; the compose job is a pure deployment change that should land last, once the build is proven and the catalog is populated.
**Delivers:** v1.x features as validated (actor/director search, more-in-this-collection, trailer/TMDB/Letterboxd links, format/region/edition badges, recently-added row, wishlist mailto, `--refresh`); compose job in `patrickclery.github.io` (checkout this repo, build with `NEXT_PUBLIC_BASE_PATH=/sale/dvds`, copy into `out/sale/dvds/`), `repository_dispatch` from this repo with a least-privilege fine-grained PAT; decision on whether to disable the `/dvd-seller/` Pages site or keep it as preview; `out/` size gate. Reserve the `NEXT_PUBLIC_IMAGE_MODE=local` flag and `cache-images.ts` for when there is a reason.
**Avoids:** Pitfall 3 (cross-repo permissions, stale builds, duplicate CNAME), Pitfall 7 (image cache bloat).

### Phase Ordering Rationale

- The zod schema is the single dependency of everything else; both the SPA and the scripts import it, so it must be first and must be treated as a contract, not an implementation detail.
- Splitting scripts (Phase 4) from the skill (Phase 5) makes the matcher testable without Claude and without photos, which is the only way to measure precision on the remake golden set before trusting it on 300 spines.
- Deploying early on topology (a) de-risks the mount: (b) becomes a 15-line YAML change with no application code touched, because every URL already goes through `next/link` or `withBasePath()`.
- Polish is deferred until real data exists because filter defaults, cross-sell intersections and box-set handling are all data-dependent.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 5 (Ingestion skill):** vision prompt design for rotated, glare-prone, many-spines-per-photo input is the least documented part of the project; only community and paper sources. Plan a spike with 3-5 real shelf photos before committing the SKILL.md rubric. Also verify current Claude Code skill frontmatter (`allowed-tools`, `disable-model-invocation`, `${CLAUDE_SKILL_DIR}`) against live docs at planning time.
- **Phase 6 (Mount at /sale/dvds):** cross-repo compose has known permission/staleness traps; PITFALLS flags it as the phase most likely to need its own spike (PAT scopes for `repository_dispatch`, whether to build from source vs. publish an `out/` branch, disabling vs. keeping the `/dvd-seller/` site).
- **Phase 1 (schema, licensing sub-item only):** if any doubt remains about the non-commercial posture, a short email to sales@themoviedb.org resolves it; otherwise proceed on the staff-forum reading.

Phases with standard patterns (skip research-phase):
- **Phase 2 (SPA skeleton):** Next.js static export, `generateStaticParams`, `basePath`, Suspense + `useSearchParams` are all official-docs territory and mirrored by the sibling repo and the `nextjs/deploy-github-pages` template.
- **Phase 3 (Deploy a):** verbatim copy of a workflow already running in production.
- **Phase 4 (Ingestion scripts):** TMDB/OMDb endpoints, `append_to_response`, throttling and disk caching are fully specified in STACK.md and ARCHITECTURE.md.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM-HIGH | Versions and peer ranges verified against the npm registry; prior art verified via GitHub API; provider terms from official pages. Licensing interpretation is a judgment call. |
| Features | MEDIUM | Cross-corroborated across Letterboxd, Plex, Libib, eBay, tinyMediaManager and TMDB docs; no single-source claims, but no user validation yet. |
| Architecture | MEDIUM-HIGH | Every load-bearing claim (basePath behaviour, Suspense build error, export layout, Pages custom-domain inheritance) has two official sources plus live CDN measurements; sibling repo read directly. |
| Pitfalls | MEDIUM | Static-export and Pages mechanics from official docs; TMDB collision behaviour from community threads and one OSS fix PR; vision failure modes from literature, not from this project's photos. |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- **TMDB commercial-use grey area:** research relied on staff forum posts, not a written ruling. Handle by recording the non-commercial posture in PROJECT.md in Phase 1 and optionally emailing TMDB; keep prices displayed but no checkout/ads.
- **Real-world vision accuracy on Patrick's shelves:** unmeasured. Handle with a Phase 5 spike on real photos and the golden test set; keep the rotate-and-reread fallback in reserve.
- **Whether to show prices at all** given the licensing posture: FEATURES treats price as table stakes, PITFALLS suggests "contact seller, no prices" as the safest reading. Decide in Phase 1; research leans toward showing price with "Ask" fallback and no payment path.
- **Rating source when OMDb key is absent:** design supports both; confirm at Phase 1 which the UI labels as default (recommendation: OMDb IMDb rating when present, TMDB rating clearly labelled otherwise).
- **`/dvd-seller/` vs `/sale/dvds` canonical URL after Phase 6:** decide whether to disable the project Pages site or keep it as a preview target; avoid two live canonical URLs.
- **Next 16 vs sibling's Next 15:** greenfield on 16 is recommended; if the two repos are ever composed at the build level (shared lockfile), revisit.
- **GitHub search recall:** MEDIUM confidence that no niche static catalog was missed; acceptable because the differentiating half (photo ingestion) is absent from the space regardless.

## Sources

### Primary (HIGH confidence)
- npm registry (`npm view` 2026-09-21): next 16.3.5, react 19.3.0, tailwindcss 4.3.3, typescript 5.9.3, zod 4.6.5, lucide-react 1.47.0, fuse.js 7.5.0, p-throttle 8.1.1, sharp 0.35.4
- GitHub API (`gh api repos/...`, `gh search repos`, 2026-09-21): all prior-art stars, push dates, licences, archived flags
- Sibling repo `~/github.com/patrickclery/patrickclery.github.io` (`next.config.ts`, `deploy.yml`, `public/.nojekyll`, `public/CNAME`): conventions read directly
- Live measurement: `curl -I https://image.tmdb.org/t/p/w342/...` (63-91 KB, `cache-control: public, max-age=31536000`)

### Secondary (MEDIUM confidence; official docs via webfetch, cross-verified)
- Next.js docs: static exports guide, `basePath`, `generateStaticParams`, `useSearchParams` Suspense requirement, installation requirements
- `nextjs/deploy-github-pages` template and `actions/configure-pages` source (basePath injection)
- GitHub Pages docs: limits (1 GB, 100 GB/mo), custom 404, custom-domain inheritance for project sites
- TMDB: rate limiting, API terms of use, FAQ (commercial definition, 6-month cache), attribution/logos, image basics, `append_to_response`, `search/movie`, staff forum posts on free public non-monetized sites
- OMDb: key page (1,000/day, Poster API patrons-only, CC BY-NC 4.0)
- IMDb: `data.imdb.com` (AWS Data Exchange only), non-commercial datasets, Conditions of Use
- Claude Code skills docs: frontmatter, `${CLAUDE_SKILL_DIR}`, `${CLAUDE_PROJECT_DIR}`, `allowed-tools`, plugin packaging
- Letterboxd, Plex, Libib, eBay item-specifics, tinyMediaManager feature pages (feature benchmarks)

### Tertiary (LOW confidence; needs validation)
- Community threads: `.nojekyll` for `_next/`, cross-repo `download-artifact` permissions, Git LFS not served by Pages, TMDB image CDN connection limits
- GMDB PR #84 (server-side `primary_release_year` fix) and TMDB talk threads on search ordering
- Spine-recognition literature (YOLOv11 + PaddleOCR paper) and bookshelf-scanner OSS projects: failure modes under tilt/glare
- npmtrends / blog comparisons of MiniSearch vs Fuse.js (non-critical optional choice)

---
*Research completed: 2026-09-21*
*Ready for roadmap: yes*
