# Pitfalls Research

**Domain:** Static, zero-hosting DVD-collection catalog SPA (Next.js static export on GitHub Pages) with agent-driven ingestion from photos of DVD spines via TMDB/OMDb metadata
**Researched:** 2026-09-21
**Confidence:** MEDIUM overall. Static-export and GitHub Pages mechanics verified against official docs (Next.js 16.3.5 docs, GitHub Pages limits doc, sibling repo config). TMDB/OMDb terms quoted from official pages but licensing interpretation for a "for sale" catalog is a judgment call. Vision/disambiguation pitfalls are drawn from TMDB community threads, one OSS fix PR, and spine-recognition literature — MEDIUM.

Phase names used below: **Data/schema**, **SPA**, **Ingestion skill**, **Deploy/mount**.

## Critical Pitfalls

### Pitfall 1: The free TMDB/OMDb licenses are non-commercial — and this site exists to sell DVDs

**What goes wrong:**
The project assumes "free API tier" means free for this use. TMDB's API Terms grant a non-commercial license only; their FAQ defines a project as commercial "if the primary purpose is to create revenue for the owner" and directs commercial users to sales@themoviedb.org for a written agreement. OMDb's data is CC BY-NC 4.0 (non-commercial) and its Poster API is patrons-only. A catalog whose stated purpose is selling physical goods sits in a grey zone that could be read as commercial.

**Why it happens:**
Developers read "free API" and stop. The TMDB terms list examples of commercial use (charging for apps, subscription content, driving revenue on destination sites) that don't obviously match "a hobbyist listing used DVDs", so nobody checks.

**How to avoid:**
- Treat this as a decision, not an oversight: record in PROJECT.md whether the site is positioned as a personal, non-commercial catalog (contact-the-seller, no prices/checkout, no ads) or a commercial storefront. Keep the former unless volume grows.
- Ship the exact required notice in the footer/About: *"This website uses TMDB and the TMDB APIs but is not endorsed, certified, or otherwise approved by TMDB."* plus an approved TMDB logo that is less prominent than the site's own branding.
- If OMDb is used for IMDb ratings, add a CC BY-NC attribution line as well. Prefer TMDB's own `vote_average` as the displayed rating to reduce license surface; store `imdb_id` and link out rather than reproducing IMDb data.
- Respect TMDB's cache rule: data may not be cached longer than 6 months. Store `fetchedAt` per entry and have the skill offer a `--refresh-stale` pass.
- Never scrape imdb.com. IMDb Conditions of Use prohibit "data mining, robots, screen scraping" without written consent, and the official datasets may not be used to build an online movie database.

**Warning signs:**
No attribution component in the layout; entries have no `fetchedAt`/`source` fields; anyone proposes adding a "Buy now"/Stripe link or ads; a skill step fetches `imdb.com/title/...` HTML.

**Phase to address:**
Data/schema (add `source`, `fetchedAt`, `imdbId`; decide displayed-rating source). SPA (attribution component in root layout). Ingestion skill (no IMDb scraping path exists at all).

---

### Pitfall 2: Static export under `/sale/dvds` breaks in five different ways at once

**What goes wrong:**
The site works at `localhost:3000` and at `patrickclery.github.io/dvd-seller/`, then mounted under `patrickclery.com/sale/dvds` it has broken CSS, 404 on refresh of a detail page, images pointing at `/posters/x.jpg` instead of `/sale/dvds/posters/x.jpg`, or a hard build error about Suspense.

**Why it happens:**
Five independent mechanisms, each with its own footgun:
1. **`basePath` is inlined at build time** and auto-applied to `next/link` and route navigation only. It is NOT applied to `next/image` `src`, to plain `<img src="/...">`, to `fetch("/data/catalog.json")`, to `<link rel="icon">`, or to CSS `url()`. Every public-asset reference needs the prefix manually.
2. **Dynamic routes** (`/movie/[slug]`) require `generateStaticParams()` under `output: 'export'`; `dynamicParams: true` is unsupported. Forgetting this either fails the build or silently emits nothing for detail pages.
3. **`trailingSlash: true`** (as the sibling repo uses) emits `/movie/slug/index.html`, which GitHub Pages serves on a direct hit. Without it, `/movie/slug.html` is emitted and a deep link to `/movie/slug` 404s on Pages. Keep `trailingSlash: true` and never hand-write hrefs without the trailing slash.
4. **`useSearchParams()` must be wrapped in `<Suspense>`** or `next build` fails with `missing-suspense-with-csr-bailout` — and it commonly surfaces on the `/404` route because the root layout or a shared filter bar reads URL state. Filters-in-the-URL (`?genre=horror&sort=rating`) is the natural design for this app, so this will bite.
5. **`.nojekyll` must exist in `out/`.** GitHub Pages runs Jekyll by default, which ignores underscore-prefixed directories, so `_next/` returns 404 and the page renders unstyled. The sibling repo already has `public/.nojekyll`; copy it.

Also: GitHub Pages is **case-sensitive**. A slug generated as `The-Thing-1982` and linked as `the-thing-1982` is a 404 in prod and works on a macOS/Windows dev filesystem.

**How to avoid:**
- Read `basePath` from one env var (`NEXT_PUBLIC_BASE_PATH`), set `basePath` and `assetPrefix` from it in `next.config.ts`, and export a single `withBase(path)` helper used for every non-`Link` URL (poster fallbacks, JSON fetch, favicon, OG image).
- Prefer importing `catalog.json` at build time in Server Components over client-side `fetch` — no runtime path to get wrong, and the grid is fully prerendered.
- Lowercase all slugs at generation time; add a unit test that every `slug` matches `/^[a-z0-9-]+$/` and is unique.
- Put every `useSearchParams` consumer in a small client component wrapped in `<Suspense fallback={...}>`; keep the page shell static.
- Keep a **matrix build in CI**: build once with `NEXT_PUBLIC_BASE_PATH=""` and once with `/sale/dvds`, then run a link checker (e.g. `lychee` or a 30-line script over `out/`) that asserts every `href`/`src` inside `out/` resolves to a file under `out/` after stripping the prefix.
- Verify `out/.nojekyll` exists in the workflow before upload.

**Warning signs:**
Any `src="/..."` string literal in a component; `fetch('/...')` in client code; `[slug]/page.tsx` without `generateStaticParams`; build error mentioning Suspense/CSR bailout; a slug helper that doesn't call `.toLowerCase()`.

**Phase to address:**
SPA (routing, Suspense, slug rules, `withBase`). Deploy/mount (matrix build, link check, `.nojekyll`).

---

### Pitfall 3: Two repos, one domain — the mount strategy has hidden permission and staleness traps

**What goes wrong:**
Option (a) from PROJECT.md — the main site's workflow pulls `dvd-seller`'s `out/` into `out/sale/dvds/` — fails with `Resource not accessible by integration`, or silently ships a build from last month, or the DVD site's own Pages deployment collides with the main site's custom domain.

**Why it happens:**
- The default `GITHUB_TOKEN` cannot read another repo's artifacts; cross-repo `actions/download-artifact` needs `actions: read` on the *source* repo via a PAT or fine-grained token passed as `github-token`. Once a workflow declares a `permissions:` block (the sibling does), everything not listed is denied.
- Artifacts expire (default 90 days); if the main site rebuilds after the DVD artifact is gone, the pull step fails or falls back to a stale checked-in copy.
- A push to `dvd-seller` does not rebuild `patrickclery.github.io`, so the live catalog lags until something else triggers the main site — the seller adds 20 DVDs and nothing changes.
- GitHub allows a custom domain (`patrickclery.com`) in exactly one repo's CNAME. If `dvd-seller` also has a `public/CNAME` with `patrickclery.com`, one of the two Pages sites breaks. And because the user site has a custom domain, `dvd-seller`'s own project Pages URL is already `patrickclery.com/dvd-seller/` (project sites are served under the user site's custom domain) — that is a second, unintended public URL with the wrong `basePath`.
- Git submodules for this cause detached-HEAD confusion and require a `submodules: true` checkout plus a manual bump commit for every catalog change.

**How to avoid:**
- Pick the simplest composition: have `dvd-seller` push its built `out/` to a `gh-pages`-style branch (or a release asset), and have the main site's workflow `git clone --depth 1` that branch into `out/sale/dvds/` at build time. Public repo + public branch means no token dance.
- Add a `repository_dispatch` step at the end of `dvd-seller`'s workflow that triggers the main site's deploy (needs one fine-grained PAT with `contents: write`/`actions: write` on the main repo stored as a secret in `dvd-seller`). Document the secret name in the README.
- Do **not** put a CNAME in `dvd-seller`. Either disable Pages on `dvd-seller` entirely (once composition works) or accept that `patrickclery.com/dvd-seller/` exists and build that deployment with `basePath=/dvd-seller`. Don't leave it broken.
- Avoid submodules.

**Warning signs:**
`Resource not accessible by integration` in Actions logs; `permissions:` block without `actions: read` while using `download-artifact` with `repository:`; a `public/CNAME` in `dvd-seller`; a `.gitmodules` file; the catalog JSON changed but `patrickclery.com/sale/dvds` didn't.

**Phase to address:**
Deploy/mount. This is the phase most likely to need its own research spike; decide (a) vs (b) subdomain here.

---

### Pitfall 4: Search-by-title returns the popular remake, and the skill commits the wrong film silently

**What goes wrong:**
Spine says "THE THING". TMDB `/search/movie?query=The Thing` returns the 1982 Carpenter film first; the DVD is the 2011 prequel — or vice versa. Same for *Halloween* (1978/2007/2018), *Dune*, *Total Recall*, *RoboCop*, *Ghostbusters*, *Planet of the Apes*, *Godzilla*, *Carrie*, *Suspiria*, *Oldboy*. The catalog now shows a wrong poster, wrong cast, wrong year, and the buyer who wanted the 2011 film receives the 1982 one — or complains the listing was wrong.

**Why it happens:**
TMDB's text search is relevance/popularity ranked and returns 20 per page. Filtering by year *client-side* on that first page fails when the intended film is crowded out entirely (documented fix in GMDB PR #84: apply `primary_release_year` server-side, fall back to a plain query only when the year-scoped search is empty). DVD spines often carry no year, and vision reads are lossy, so the skill is tempted to take result #1.

**How to avoid:**
- Search strategy in the skill: (1) if the spine or the seller supplies a year, query with `primary_release_year` first; (2) otherwise, fetch results and detect **title collisions** — if two or more results share a normalized title, treat as ambiguous regardless of popularity; (3) use spine cues (studio logo, "Collector's Edition", actor names, rating badge, aspect of artwork) to bias, but never to auto-resolve a collision.
- Confidence gate: auto-accept only when exactly one result has normalized-title match AND no other result within a Levenshtein distance of 2 exists in the top 10. Everything else goes to a **confirmation queue** the seller answers in one batched prompt ("3 ambiguous — pick: 1a The Thing (1982) / 1b The Thing (2011) / skip").
- Persist the seller's choice (`tmdbId`, `confirmedBy: "seller"`) so re-runs never re-ask.
- Handle non-movies: `/search/multi` or a TV fallback for box sets ("Friends: Season 3", "The Office Complete Series") — write them as `type: "tv"` with `seasonNumbers`, not as a movie guess.
- Foreign editions: search with the read title, then fall back to `original_title` matching and the `language` hint from the spine; cover art from a German release will still resolve to the same `tmdbId`.
- Sequels with roman numerals/subtitles: normalize "II"→"2", strip "The", collapse punctuation before comparing, but display TMDB's canonical title.

**Warning signs:**
Skill code that does `results[0]` anywhere; no `year` field in the confirmation prompt; test fixtures that contain no remake titles; a run of 50 photos that produced zero confirmation prompts (too confident, not too good).

**Phase to address:**
Ingestion skill. Seed a **golden test set** of at least 15 known-hard spines (remakes, sequels, box sets, foreign edition, glare, partial title) before writing the matcher; measure precision on it.

---

### Pitfall 5: Re-running ingestion duplicates entries or rewrites every slug

**What goes wrong:**
The seller re-photographs a shelf after moving DVDs around; the skill appends 40 duplicates. Or a slug algorithm tweak changes `/movie/the-thing/` to `/movie/the-thing-1982/`, breaking every shared link and every browser bookmark.

**Why it happens:**
Identity is derived from the vision read (title string) instead of the metadata provider's stable ID. Slugs are generated from mutable data (title + whatever disambiguator was in fashion). The skill and the SPA each have their own idea of the schema, and neither validates.

**How to avoid:**
- **Primary key is `tmdbId`** (integer), with `imdbId` as a secondary stable ID. Dedup on `tmdbId` before append; a second physical copy of the same film is `copies: 2` or a `copies[]` array with per-copy condition/price, not a second entry.
- **Slug is generated once and frozen**: `${kebab(title)}-${year}` on first insert, stored in the JSON, never recomputed. Uniqueness is enforced at write time; collisions get a `-2` suffix. Add a test that slugs in the committed file are unique and match the regex.
- **One schema, one source of truth**: define the catalog entry with Zod (or JSON Schema) in a shared package/`schema/` folder imported by both the skill's writer and the SPA's loader. The SPA build fails fast on invalid data; the skill refuses to write invalid entries. Include a `schemaVersion` at the file root.
- Sort entries deterministically (by `tmdbId`) and pretty-print with stable key order so git diffs show only real changes and PR review is possible.
- Split data if the single file gets unwieldy: `catalog/index.json` (grid fields only: id, slug, title, year, poster, rating, genres, saleState) plus `catalog/movies/<tmdbId>.json` (cast, synopsis). At a few hundred titles a single file is fine (~200 KB); revisit past ~1,000 or when the git diff noise hurts.

**Warning signs:**
Two entries with the same `tmdbId`; a slug function called from both the skill and a React component; the JSON diff for "added 3 DVDs" touches 300 lines; no `schemaVersion` key.

**Phase to address:**
Data/schema first (schema package, ID and slug rules, dedup contract), before SPA or skill code.

---

### Pitfall 6: Secrets, raw photos, and EXIF GPS end up in a public repo

**What goes wrong:**
`TMDB_API_KEY` is put in `NEXT_PUBLIC_*` or a `.env` that gets committed; the ingestion photos (often 3–8 MB each, with GPS coordinates of the seller's home) are committed alongside the catalog "for provenance"; the repo balloons and doxxes the seller.

**Why it happens:**
"The site needs the key to show posters" — it doesn't; image URLs on `image.tmdb.org` are unauthenticated, and all metadata is baked in at build time. Photos get committed because the skill was run inside the repo and `git add -A` was used.

**How to avoid:**
- The SPA must have **zero** `NEXT_PUBLIC_*` secrets and make zero API calls at runtime. Enforce with a CI grep for `api_key=` and `NEXT_PUBLIC_TMDB` in `out/`.
- The skill reads `TMDB_API_KEY` (and optional `OMDB_API_KEY`) from the environment only, fails with a clear message if missing, and never writes them to any file.
- `.gitignore` from day one: `photos/`, `*.jpg`, `*.jpeg`, `*.heic`, `*.png` (except `public/`), `.env*`. Skill accepts photos from **any path** (default `~/Pictures/dvd-spines` or CLI arg), never from inside the repo.
- If any image is ever committed (local cache mode, README screenshots), strip metadata: `exiftool -all= -overwrite_original` as a pre-commit hook or skill step. Poster/headshot images fetched from TMDB have no personal EXIF but strip anyway for consistency.
- Add a secrets scanner (`gitleaks` action) to the workflow.

**Warning signs:**
Any `NEXT_PUBLIC_*_KEY`; `git status` showing `*.jpg` outside `public/`; repo size jump in the hundreds of MB; a `photos/` folder in the tree.

**Phase to address:**
Data/schema (gitignore, repo hygiene, CI grep) and Ingestion skill (env-only keys, photo path outside repo).

---

### Pitfall 7: Opt-in local image cache bloats the repo past what GitHub Pages will serve

**What goes wrong:**
Self-contained mode downloads `w500` posters (~40–80 KB each) plus `w185` headshots for a 10-person cast per film (~10 × 8 KB). At 500 titles that is roughly 25–40 MB of posters and 40 MB of headshots — tolerable once, but every re-fetch at a new size or every replaced poster adds to git history forever. Someone reaches for Git LFS; **GitHub Pages does not serve LFS objects** (pointer files get deployed instead), and the LFS free quota is 1 GB storage / 1 GB bandwidth per month.

**Why it happens:**
"Fully self-contained" sounds like a checkbox. Git's immutability means image churn is permanent, and the 1 GB published-site limit plus 100 GB/month soft bandwidth are not top of mind.

**How to avoid:**
- Keep on-demand CDN as the default (already decided). For cache mode, fetch at **one** fixed size per asset class (`w342` posters, `w185` profiles), name files by content-stable ID (`posters/<tmdbId>.jpg`), and never re-fetch an existing file.
- Store cached images **outside git history**: generate them in CI during build (fetch into `out/img/` at build time; ~600 requests at 20 concurrent is well under a minute and within TMDB's ~40 req/s ceiling) rather than committing them. The committed artifact stays small; the deployed site is self-contained.
- If they must be committed (offline builds), put them on an orphan branch or a separate `dvd-seller-assets` repo consumed at build time, so `main` history stays lean. Document the `git gc` implication.
- Do not use Git LFS for anything served by Pages.

**Warning signs:**
`git count-objects -vH` growing tens of MB per ingestion run; `.gitattributes` with `filter=lfs`; poster files with size-in-name (`-w500`, `-w780`) side by side.

**Phase to address:**
Data/assets (define cache-mode design as build-time fetch, not commit). Deploy/mount (bandwidth/size check in CI: fail if `out/` > 500 MB).

---

### Pitfall 8: Vision read of spines is wrong in ways that look right

**What goes wrong:**
The model returns a confident, plausible title that is not what's on the spine: "Alien" for "Aliens", "Die Hard 2" for "Die Hard" because the "2" from the neighboring spine bled in, "Fast & Furious" for "2 Fast 2 Furious", "Special Edition" absorbed into the title, a partially occluded "…of the Rings" resolved to the wrong volume, a distributor name ("Criterion", "Lionsgate") read as the title, or the same spine counted twice across two overlapping photos.

**Why it happens:**
Spine text is rotated 90°, in mixed stylized fonts, under glare, often with 15–30 spines per photo at 40–80 px height each. Spine-recognition literature shows detection degrades with tilt and dim backgrounds, and off-the-shelf OCR is unstable on natural-scene text; an LLM fills gaps from priors, so errors are fluent rather than garbled. Adjacent-spine bleed is the specific failure of many-spines-per-photo input.

**How to avoid:**
- Prompt the vision step for a **structured per-spine list** — `{ index, rawText, titleGuess, yearGuess, editionCues, confidence, notes }` — rather than "list the movies". Ask it to keep `rawText` verbatim and to mark unreadable spines rather than guess.
- Optionally pre-rotate the image 90° (spine text reads bottom-to-top on most US releases) and, for high-density shelves, tile the photo into 2–3 overlapping crops; dedupe by `tmdbId` afterwards, which also handles overlapping photos.
- Two-signal acceptance: vision `confidence >= 0.8` **and** TMDB match unambiguous (Pitfall 4). Otherwise queue for confirmation. Show the seller the crop/`rawText` next to the candidate so confirmation takes two seconds.
- Provide a `--review` mode that prints the full accepted list with year and poster URL before writing, and a `--dry-run` that writes nothing.
- Track a per-run **acceptance rate**; if >95% of spines auto-accept on a messy photo, the gate is too loose.

**Warning signs:**
Vision output as free text; no `rawText` field kept for audit; catalog entries whose `title` contains "Edition", "Collection", "Widescreen", a studio name, or a stray digit; zero confirmations across a full shelf.

**Phase to address:**
Ingestion skill; the golden test set from Pitfall 4 should include glare, adjacency, and partial-occlusion photos.

---

### Pitfall 9: The skill only works on Patrick's machine

**What goes wrong:**
The skill hard-codes `~/github.com/patrickclery/dvd-seller/data/catalog.json`, assumes `zsh`, assumes `jq`/`exiftool`/`node 20` are installed, reads the key from `~/.config/...`, and writes with Patrick's key ordering. Another seller installs it and gets a stack trace.

**Why it happens:**
It's written and tested in one environment; portability is an afterthought.

**How to avoid:**
- Use `${CLAUDE_SKILL_DIR}` for bundled scripts and `${CLAUDE_PROJECT_DIR}` for repo-relative paths in SKILL.md; never a literal home path.
- Resolve the catalog path in this order: CLI arg → `DVD_CATALOG_PATH` env → `./data/catalog.json` relative to `${CLAUDE_PROJECT_DIR}` → prompt the user. Print the resolved path at start.
- Keys from env only (`TMDB_API_KEY`), with a one-line message pointing to where to get one. No fallback to a file.
- Bundle the writer as a small Node/TypeScript script (the repo already needs Node) with **no non-npm dependencies**; if `exiftool` is optional, detect and skip with a warning.
- Ship `--dry-run`, `--review`, and `--photos <dir|file...>` flags; the SKILL.md example should be runnable on a fresh clone.
- Put a "Install for another seller" section in README and test it in a clean temp directory in CI (clone, set env, run `--dry-run` against fixture photos).

**Warning signs:**
`/home/patrick` or `patrickclery` in any skill file; `allowed-tools` listing an absolute path; no `--dry-run`; README says "run the skill" without listing prerequisites.

**Phase to address:**
Ingestion skill; portability test in CI belongs in Deploy/mount.

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Client-side `fetch('/data/catalog.json')` instead of build-time import | Feels "dynamic", easy | Path breaks under `basePath`, slow first paint, no prerendered grid, SEO-less detail pages | Never for the primary catalog; fine for an optional "last updated" ping |
| Take TMDB `results[0]` | Zero-friction ingestion | Silent wrong films in the catalog, buyer trust loss, manual audit later | Never |
| Slug = kebab(title) with no year | Pretty URLs | Collisions on remakes, forced slug migration later | Never; use `title-year` from day one |
| Single giant `catalog.json` | Simplest possible data layer | Noisy diffs, larger hydration payload | Fine up to ~1,000 entries; plan the index/detail split but don't build it yet |
| Commit cached posters to `main` | Self-contained checkout | Permanent history bloat, approaches Pages 1 GB | Only on an orphan/assets branch, or never — prefer build-time fetch |
| Skip the Suspense wrapper by making the page `'use client'` | Build passes | Whole grid becomes CSR, blank first paint, hydration flicker | Never |
| Store IMDb rating from OMDb without attribution | One more number on the card | License exposure (CC BY-NC) | Only with attribution and a non-commercial posture; otherwise show TMDB rating |
| Second Pages deployment of `dvd-seller` left enabled with wrong basePath | Preview URL | Broken public URL at `patrickclery.com/dvd-seller/` | Only during development; disable or fix before sharing links |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| TMDB API | Assume "free" covers a sales catalog; skip attribution | Non-commercial posture, exact attribution notice + approved logo in footer, `fetchedAt` for the 6-month cache rule |
| TMDB API | Hit search with no year and take first result | `primary_release_year` first when known; detect title collisions; queue ambiguities |
| TMDB API | Burst hundreds of parallel requests during ingestion | Throttle to ~10–20 req/s with a queue; back off on 429; batch by photo |
| TMDB API | Ship `TMDB_API_KEY` in `NEXT_PUBLIC_*` | Key is ingestion-only; SPA makes zero API calls |
| `image.tmdb.org` | Hotlink the `original` size for grid thumbnails | Use `w185`/`w342` for grid, `w500` for detail; cap concurrent loads (browser does this, but don't eager-load 500 images). Hotlinking is permitted in practice for API users; TMDB forbids using it as generic image hosting — only load images tied to catalog entries |
| OMDb | Rely on it for posters | Poster API is patrons-only; use TMDB for images, OMDb only for the IMDb rating if at all; 1,000/day is enough for a few hundred titles but not for a nightly refresh of everything |
| IMDb | Fetch anything from imdb.com | Only link out via `https://www.imdb.com/title/<imdbId>/`; never scrape |
| GitHub Pages | Forget `.nojekyll` | Copy `public/.nojekyll` from the sibling; assert it's in `out/` in CI |
| GitHub Pages | Second CNAME with `patrickclery.com` | No CNAME in `dvd-seller`; compose into the main site's `out/sale/dvds/` |
| GitHub Actions | Cross-repo `download-artifact` with default token | Clone a published branch of `dvd-seller` instead; or pass a fine-grained PAT with `actions: read`; add `repository_dispatch` to trigger main-site rebuilds |
| Claude Code skill | Absolute paths, keys in files | `${CLAUDE_SKILL_DIR}`, `${CLAUDE_PROJECT_DIR}`, env-only keys, `--dry-run` |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Rendering all posters eagerly | Slow first paint on mobile; 300 image requests at once | `loading="lazy"` + `decoding="async"` on every `<img>` below the fold; only first ~12 eager; fixed `aspect-ratio: 2/3` containers | Noticeable at ~100 titles on 4G |
| Layout shift from unsized images | Grid jumps as posters load; CLS penalty; mis-taps | Explicit `width`/`height` or CSS aspect-ratio box; poster placeholder color/blur | Any size |
| Hydrating the full catalog (cast, synopsis) into the grid page | Large HTML/JS payload; slow TTI | Grid page consumes a slim index projection; detail pages get full entries via `generateStaticParams` | ~500+ titles with full cast |
| Filtering with URL state read on first render | Hydration mismatch warnings; flash of unfiltered grid | Read `useSearchParams` in a Suspense'd client component; render the unfiltered static grid as fallback | Any size, appears immediately |
| One `catalog.json` imported by every route | Every detail page bundles the whole file | Import the index in the grid, and per-movie JSON (or a `Map` lookup in a Server Component) in detail pages | ~1,000 titles |
| Ingestion fetching credits + details per movie serially | 500 titles × 3 calls × 300 ms = 8 minutes | `append_to_response=credits,external_ids` collapses to 1 call/movie; concurrency 10 | Any run over ~50 titles |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| API key in `NEXT_PUBLIC_*` or committed `.env` | Key abuse, TMDB account suspension | Ingestion-only env var; CI grep of `out/` for `api_key` |
| Committing spine photos | GPS EXIF reveals seller's home address publicly | Photos live outside the repo; `.gitignore` image globs; `exiftool -all=` on anything committed |
| Seller contact details as plain `mailto:` in HTML | Scraped for spam | Obfuscate lightly or use a contact form service the seller already has; at minimum a `mailto:` with subject prefill is acceptable for an unlisted page |
| Prices/condition notes leaking personal context | Minor privacy | Keep `notes` field seller-private (not rendered) vs `publicNotes` |
| Deploy workflow with broad `permissions` plus a PAT | PAT leak via logs | Least-privilege fine-grained PAT scoped to one repo; never `echo` secrets |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| No sold/reserved state, or sold items vanish | Buyer asks about a sold DVD; or link they were sent now 404s | `saleState: available \| reserved \| sold`; keep sold items in the catalog with a badge and a "Sold" filter default-off |
| No contact path | Buyer finds the film and can't act — the whole funnel dies | Persistent "Interested? Email/DM" CTA on grid and detail; prefilled subject with title + year |
| Filters lost on back navigation | Buyer opens a detail, hits back, sees the full unfiltered grid | Filters in URL query string (with Suspense); `Link` preserves search params |
| Deep link to a detail page 404s | Shared links die | `trailingSlash: true`, `generateStaticParams`, link checker in CI |
| Grid sorted by "popularity" with no explanation | Buyers don't know what the number means | Label as "TMDB popularity" or just use it as a hidden sort key; show rating instead |
| Rating shown as "IMDb 7.8" when it's TMDB's `vote_average` | Misleading; potential attribution issue | Label the source honestly; link to IMDb for the IMDb rating |
| Poster missing (TMDB has none) | Broken image icon | Fallback card with title/year; skill flags entries with `posterPath: null` for manual override |
| Box set shown as a single movie | Buyer confusion | Distinct `type: tv` card treatment; season list on detail |
| Tiny tap targets on phone | Mis-taps between posters | Minimum 44 px, gap ≥ 8 px, phone-first grid of 2–3 columns |

## "Looks Done But Isn't" Checklist

- [ ] **basePath:** Works at `/`, but verify a full build with `NEXT_PUBLIC_BASE_PATH=/sale/dvds` served from a subfolder locally (`npx serve out --listen 3000` under a `sale/dvds` parent dir) — check CSS, posters, favicon, JSON, deep link refresh.
- [ ] **Detail pages:** Grid links work via client navigation; verify a hard refresh on `/movie/<slug>/` returns 200 from Pages (not the SPA fallback).
- [ ] **`.nojekyll`:** Present in `out/` — check the uploaded artifact, not just `public/`.
- [ ] **Attribution:** TMDB notice and logo rendered on every page (root layout), not only on an About page nobody opens.
- [ ] **Dedup:** Re-run ingestion on the same photos — entry count must not change.
- [ ] **Slug stability:** Re-run ingestion — no slug in the diff changes.
- [ ] **Confirmation queue:** Run on the golden set — the remake pairs must prompt, not auto-resolve.
- [ ] **Secrets:** `grep -r "api_key" out/` and `grep -r "NEXT_PUBLIC_TMDB" .` return nothing.
- [ ] **Photos:** `git ls-files | grep -Ei '\.(jpe?g|heic|png)$'` returns only `public/` assets.
- [ ] **Sold state:** At least one fixture entry is `sold` and renders with a badge and is excluded by the default filter.
- [ ] **Contact CTA:** Visible on phone viewport without scrolling on the detail page.
- [ ] **Main-site composition:** Pushing to `dvd-seller` results in a new deploy of `patrickclery.com` within minutes (dispatch works), and `patrickclery.com/dvd-seller/` is either disabled or correct.
- [ ] **Skill portability:** Fresh clone in a temp dir + `TMDB_API_KEY` + `--dry-run` on fixture photos succeeds with no edits.
- [ ] **Schema:** SPA build fails when a fixture entry violates the Zod schema (prove the guard works).

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Wrong film committed | LOW | `skill fix <slug> --tmdb-id <id>`: refetch metadata, keep slug if title/year unchanged, else add redirect stub page at old slug |
| Slug migration needed | MEDIUM | Emit static `redirects.json` and a client-side redirect page at each old slug (no server redirects on Pages); keep `previousSlugs[]` in entries |
| Duplicates in catalog | LOW | One-off dedupe script by `tmdbId`, merging `copies[]`; add the test that prevents recurrence |
| Photos/secrets committed | HIGH | Rotate the API key; `git filter-repo` to purge; force-push; GitHub support to purge cached views if sensitive |
| Repo bloated by images | HIGH | `git filter-repo` to strip `public/img/`; move to build-time fetch; force-push and re-clone |
| Broken basePath in prod | LOW | Fix `withBase` usages; CI link checker prevents regression |
| Cross-repo artifact permission failure | LOW | Switch to public-branch clone; or add fine-grained PAT as `github-token` |
| TMDB 429s during ingestion | LOW | Backoff with jitter, resume from last written entry (skill must be idempotent) |
| License challenge from TMDB | MEDIUM | Remove prices/commercial language, confirm attribution, reply to their request; or license commercially |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Non-commercial license / attribution | Data/schema (fields) + SPA (footer) | Attribution rendered in root layout; `fetchedAt` present on all entries; PROJECT.md records the non-commercial posture decision |
| basePath / trailingSlash / Suspense / `.nojekyll` / case-sensitive slugs | SPA, verified in Deploy/mount | Matrix build (`""` and `/sale/dvds`) + link checker green; slug regex test |
| Two-repo composition, CNAME, artifact permissions, stale builds | Deploy/mount (flag for phase research) | Push to `dvd-seller` → main site redeploys; `patrickclery.com/sale/dvds/movie/<slug>/` hard-refresh 200 |
| Remake/wrong-year matching | Ingestion skill | Golden set precision ≥ 95% with all remake pairs prompting |
| Duplicates / unstable slugs / schema drift | Data/schema | Idempotent re-run test; Zod schema shared by skill and SPA; build fails on bad fixture |
| Secrets / photos / EXIF in repo | Data/schema (hygiene) + Ingestion skill | CI grep and gitleaks pass; `git ls-files` image check |
| Image cache bloat / LFS | Data/assets design + Deploy/mount | `out/` size gate; no `.gitattributes` LFS filter; cache fetched at build time |
| Vision misreads | Ingestion skill | Structured per-spine output with `rawText`; acceptance-rate metric per run; golden set includes glare/adjacency |
| Skill non-portability | Ingestion skill + Deploy/mount (CI) | Clean-temp-dir dry-run job passes |
| Buyer UX (sold state, contact, filters on back) | SPA | Fixtures include sold entry; contact CTA on detail; URL-driven filters survive back navigation |

## Sources

Official documentation (verified this session):
- Next.js static export guide (v16.3.5, updated 2026-08-25) — unsupported features, `generateStaticParams`, `trailingSlash` file layout: https://nextjs.org/docs/app/guides/static-exports
- Next.js `basePath` — auto-applied to `next/link` only; images need manual prefix: https://nextjs.org/docs/app/api-reference/config/next-config-js/basePath
- Next.js "Missing Suspense boundary with useSearchParams": https://nextjs.org/docs/messages/missing-suspense-with-csr-bailout
- GitHub Pages limits (1 GB site, 100 GB/month soft bandwidth): https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits
- GitHub Pages custom domains (one CNAME per domain; project sites served under user-site custom domain): https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site
- TMDB rate limiting (legacy 40/10s disabled 2019-12-16; ~40 req/s ceiling; respect 429): https://developer.themoviedb.org/docs/rate-limiting
- TMDB API Terms of Use (non-commercial license, 6-month cache limit, attribution notice, no image hosting): https://www.themoviedb.org/api-terms-of-use
- TMDB FAQ (commercial = primary purpose is revenue; contact sales@themoviedb.org): https://developer.themoviedb.org/docs/faq
- TMDB logos & attribution: https://www.themoviedb.org/about/logos-attribution
- OMDb API key page (FREE: 1,000 daily limit; Poster API patrons-only; CC BY-NC 4.0 footer): https://www.omdbapi.com/apikey.aspx
- IMDb Conditions of Use (no data mining/scraping without consent): https://www.imdb.com/conditions/
- IMDb "Can I use IMDb data in my software?": https://help.imdb.com/article/imdb/general-information/can-i-use-imdb-data-in-my-software/G5JTRESSHJBBHTGX
- Claude Code skills docs (`${CLAUDE_SKILL_DIR}`, `${CLAUDE_PROJECT_DIR}`, `allowed-tools`): https://code.claude.com/docs/en/skills
- Sibling repo config inspected locally: `~/github.com/patrickclery/patrickclery.github.io` (`next.config.ts` with `output: "export"`, `trailingSlash: true`, `images.unoptimized: true`; `public/.nojekyll`; `public/CNAME`; `deploy.yml` with `permissions: contents: read, pages: write, id-token: write`)

Community / issue-tracker sources (MEDIUM–LOW confidence):
- Git LFS on GitHub Pages — no plans to support: https://github.com/orgs/community/discussions/50337
- `.nojekyll` needed for `_next/`: https://github.com/vercel/next.js/issues/2029 and https://github.com/vercel/next.js/discussions/50234
- Cross-repo artifact download requires `actions: read` / explicit `github-token`: https://github.com/actions/download-artifact/issues/489 and https://github.com/orgs/community/discussions/106300
- TMDB title collision fix via server-side `primary_release_year` (GMDB PR #84): https://github.com/gvgfr/GMDB/pull/84
- TMDB search ordering / wrong-result threads: https://www.themoviedb.org/talk/5684495ac3a36860e9018833 , https://www.themoviedb.org/talk/62923e7cd48cee326d1ae31c
- TMDB image CDN 20-connection limit and ~50 req/s figures (community, unverified against official doc): https://www.themoviedb.org/talk/6558fa627f054018d5168d91
- Spine recognition difficulty with tilt/glare and OCR instability (YOLOv11 + PaddleOCR paper): https://www.mdpi.com/2079-9292/14/23/4689
- EXIF/GPS in public repos and `exiftool -all=` pre-commit: https://stefaniemolin.com/articles/devx/pre-commit/exif-stripper/

---
*Pitfalls research for: static DVD-catalog SPA with photo-to-metadata ingestion*
*Researched: 2026-09-21*
