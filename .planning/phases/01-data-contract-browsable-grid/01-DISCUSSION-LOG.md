# Phase 1: Data Contract & Browsable Grid - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-09-21
**Phase:** 1-Data Contract & Browsable Grid
**Mode:** `--auto` with ADVISOR_MODE detected (USER-PROFILE.md present; vendor philosophy `pragmatic` → calibration `standard`; NON_TECHNICAL_OWNER = false, no matching signals)
**Areas discussed:** Schema shape, Seed data, Visual design & card layout, Filter/sort/search UI, URL state, Base path & static export, Seller configuration, Repo hygiene & tooling

`[--auto] Selected all gray areas.`
`[auto] Advisor research spawns skipped: no user is in the loop to choose between researched options, and every area below is already covered in depth by .planning/research/{ARCHITECTURE,STACK,FEATURES,PITFALLS}.md. Recommended options were selected directly and logged here.`

---

## Schema shape

| Option | Description | Selected |
|--------|-------------|----------|
| Adopt ARCHITECTURE.md schema as-is | Full proposed zod schema incl. `fair/poor` condition, no `seed` provenance | |
| Adopt with tweaks (recommended) | Same schema; `reserved` status kept, eBay condition vocabulary, `confirmedBy: seed`, genre names not IDs | ✓ |
| Minimal flat schema | Only fields the grid needs; add cast/ratings later | |

**Choice:** Adopt with tweaks. **Notes:** Retrofitting identity/sale fields means re-ingesting hundreds of titles; storing cast/similar/videos now is cheap and unlocks v2 features without a migration.

## Seed data

| Option | Description | Selected |
|--------|-------------|----------|
| Fetch ~10 real titles via one-off `scripts/seed.ts` with `TMDB_API_TOKEN` (recommended) | Real poster paths, real ratings, exercises the CDN path | ✓ |
| Hand-write entries with null posters | No API key needed; placeholder posters everywhere | |
| Pull Phase 3's throttled client forward | Full ingestion lib now | |

**Choice:** One-off seed script. **Notes:** Requires Patrick to add a free TMDB v4 read token to `.env.local` — recorded as an explicit human-action checkpoint for the planner. Hand-written entries would fail success criterion 1 (posters from `image.tmdb.org`).

## Visual design & card layout

| Option | Description | Selected |
|--------|-------------|----------|
| Inherit sibling dark palette + fonts, caption below poster (recommended) | Brand-consistent when mounted under patrickclery.com; mobile-safe | ✓ |
| Cinema-red/black theme, poster-only cards with hover overlay | More "store" feel; hover-only info fails on mobile | |
| Light theme with poster cards | Diverges from sibling site | |

**Choice:** Inherit sibling. **Notes:** User UX philosophy `function-first`; sibling tokens already exist.

## Filter/sort/search UI

| Option | Description | Selected |
|--------|-------------|----------|
| Sticky toolbar + chip row on desktop, bottom sheet on mobile (recommended) | Chips satisfy "three clicks"; sheet keeps mobile grid uncluttered | ✓ |
| Left sidebar filters | Desktop-first; wastes width at 360px | |
| Dropdown selects for everything | Fewer components; more taps | |

**Choice:** Toolbar + chips + mobile sheet.

## URL state

| Option | Description | Selected |
|--------|-------------|----------|
| Hand-rolled `URLSearchParams`, short keys, defaults omitted (recommended) | Zero deps; readable share links | ✓ |
| `nuqs` | Typed helpers; one more dependency | |
| Hash routing | Unnecessary — each movie gets its own exported page | |

**Choice:** Hand-rolled, `router.replace`, Suspense-wrapped consumer.

## Base path & static export

| Option | Description | Selected |
|--------|-------------|----------|
| Env-driven `basePath` + single `withBasePath()` helper, build-time JSON import (recommended) | Mirrors sibling; retargetable to `/sale/dvds` with one env var | ✓ |
| Hard-code `/dvd-seller` | Breaks Phase 5 mount | |
| Runtime `fetch` of catalog JSON | Adds a basePath-sensitive request; not needed at this scale | |

**Choice:** Env-driven + helper.

## Seller configuration

| Option | Description | Selected |
|--------|-------------|----------|
| `data/seller.json` + zod schema (recommended) | Editable by non-TS sellers; portable with the skill | ✓ |
| `src/config/site.ts` | Typed; requires code edits for other sellers | |
| Env vars | Awkward for a multi-line blurb | |

**Choice:** `data/seller.json`.

## Repo hygiene & tooling

| Option | Description | Selected |
|--------|-------------|----------|
| Next 16 / React 19 / TS 5.9 / Tailwind 4 / zod 4 / tsx / Vitest / light Prettier+ESLint (recommended) | Matches STACK.md verified versions; adds minimal lint for scripts+tests | ✓ |
| Match sibling exactly (Next 15, no lint) | Older; sibling has no tests or scripts | |

**Choice:** Stack per STACK.md with light lint.

---

## Claude's Discretion

Header/empty-state copy, chip labels, bottom-sheet implementation, Suspense fallback skeleton, test file layout, validate/writer module split.

## Deferred Ideas

v2 discovery features (actor search, similar titles, trailer links, recently added, box sets, wishlist), local image cache mode, Fuse.js, light theme, optional TMDB licensing email.
