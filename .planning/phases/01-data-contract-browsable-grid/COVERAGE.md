# API Coverage — TMDB API v3 (Phase 1: seed-time only)

> Full coverage by default. Opt-outs are explicit, reasoned decisions.
>
> Scope note: Phase 1 touches TMDB in exactly one place — `scripts/seed.ts`, a one-off, offline, token-bearing script that fills `data/catalog.json` with 12 titles (D-13..D-15). The visitor site never calls TMDB at runtime (CAT-13); it only hotlinks `image.tmdb.org` posters. OMDb is NOT integrated in this phase (optional `imdbRating` is Phase 3 ING-02) and will get its own full-coverage baseline there — opt-outs below are not carried over to it.

| capability | decision | reason |
|---|---|---|
| `GET /movie/{id}` details | INTEGRATE | title, dates, runtime, tagline, overview, genres, poster/backdrop paths, popularity, votes |
| `append_to_response=credits` (top-12 cast, director from crew) | INTEGRATE | |
| `append_to_response=external_ids` (imdb_id) | INTEGRATE | |
| `append_to_response=videos` (YouTube trailer key) | INTEGRATE | |
| `append_to_response=release_dates` (US certification) | INTEGRATE | |
| `append_to_response=similar` (similarTmdbIds) | INTEGRATE | |
| v4 read-access bearer token on v3 endpoints | INTEGRATE | |
| Image CDN URL construction (browser-side) | INTEGRATE | `https://image.tmdb.org/t/p/{size}{path}` built only in `src/lib/images.ts` (D-07) |
| `GET /search/movie` | OPT-OUT | explicitly out of scope for Phase 1 — seed uses known tmdbIds; search + remake collision detection is Phase 3 ING-01 |
| `GET /find/{external_id}` | OPT-OUT | not needed yet — no IMDb-id-first lookups; revisit in Phase 3 if the skill reads IMDb ids from spines |
| `GET /configuration` | OPT-OUT | not needed — image base and sizes (w342/w500/w185) are fixed by D-07; TMDB documents them as stable |
| `GET /genre/movie/list` | OPT-OUT | not needed — genre names arrive inline on `/movie/{id}` (D-11); only Phase 3 search results carry bare genre_ids |
| `GET /movie/{id}/images` | OPT-OUT | not needed — poster_path/backdrop_path from details suffice; no gallery UI in v1 |
| `GET /movie/{id}/recommendations` | OPT-OUT | not needed yet — `similar` already stored; "More in this collection" is v2 DISC-02 |
| `GET /movie/{id}/{keywords,reviews,translations,alternative_titles}` | OPT-OUT | not needed — no UI consumes them (REQUIREMENTS v1/v2) |
| `GET /movie/{id}/{watch/providers,lists,changes}` | OPT-OUT | not needed — no UI consumes them (REQUIREMENTS v1/v2) |
| `GET /movie/{popular,top_rated,now_playing,upcoming}` | OPT-OUT | explicitly out of scope — the catalog is the seller's physical stock, not a discovery feed |
| `GET /trending/*`, `GET /discover/movie` | OPT-OUT | explicitly out of scope — same reason: no discovery feed |
| `GET /person/{id}` (+ images, credits) | OPT-OUT | not needed — headshots and character names come from credits; actor pages/search are v2 DISC-01 |
| `GET /collection/{id}` | OPT-OUT | not needed yet — box sets are v2 DISC-06 |
| `GET /tv/*`, `/search/tv`, `/search/multi` | OPT-OUT | explicitly out of scope — movies only (PROJECT.md) |
| Account, authentication, lists, rating, favorites, watchlist endpoints | OPT-OUT | explicitly out of scope — no user accounts, no writes to TMDB (REQUIREMENTS Out of Scope) |
| Rate limiting / 429 backoff / disk cache / concurrency gate | OPT-OUT | explicitly deferred — seed makes 12 sequential calls; the throttled cached client is Phase 3 ING-03 (D-15) |
