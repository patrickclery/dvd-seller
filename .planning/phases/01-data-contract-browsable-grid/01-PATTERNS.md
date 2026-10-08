# Phase 1: Data Contract & Browsable Grid - Pattern Map

**Mapped:** 2026-09-21
**Files analyzed:** 31 new files (greenfield repo — nothing modified)
**Analogs found:** 13 / 31 (all analogs live in the SIBLING repo; RESEARCH.md was absent at mapping time — patterns for no-analog files come from `.planning/research/ARCHITECTURE.md` § Patterns 1–5 and § Proposed Schema)

> **Sibling repo = analog codebase.** `/home/patrick/github.com/patrickclery/dvd-seller` contains no application code. Every analog path below is prefixed `SIBLING:` and resolves to `/home/patrick/github.com/patrickclery/patrickclery.github.io/<path>`. All sibling paths were verified git-tracked (`git ls-files`). **Copy the excerpts into this repo — never import across repos.**

## File Classification

| New File | Role | Data Flow | Closest Analog | Match Quality |
|----------|------|-----------|----------------|---------------|
| `next.config.ts` | config | — | `SIBLING:next.config.ts` | exact (+ basePath) |
| `package.json` | config | — | `SIBLING:package.json` | role-match (add scripts/deps) |
| `tsconfig.json` | config | — | `SIBLING:tsconfig.json` | exact |
| `postcss.config.mjs` | config | — | `SIBLING:postcss.config.mjs` | exact |
| `.gitignore` | config | — | `SIBLING:.gitignore` lines 86-113 | role-match (add D-34 entries) |
| `.nvmrc`, `.prettierrc`, `eslint.config.mjs`, `vitest.config.ts` | config | — | none (sibling has no linter/tests) | no analog |
| `public/.nojekyll` | config | — | `SIBLING:public/.nojekyll` (empty file) | exact |
| `src/app/globals.css` | config (design tokens) | — | `SIBLING:src/app/globals.css` | exact |
| `src/app/layout.tsx` | provider (root layout) | request-response | `SIBLING:src/app/layout.tsx` | exact (+ footer) |
| `src/app/page.tsx` | route (server page) | request-response → transform | `SIBLING:src/app/page.tsx` | role-match (+ Suspense, data projection) |
| `src/components/Header.tsx` | component | request-response | `SIBLING:src/components/Projects.tsx` lines 28-36 (section/heading idiom) | role-match |
| `src/components/Footer.tsx` (TMDB attribution) | component | request-response | `SIBLING:src/components/Footer.tsx` | exact |
| `src/components/MovieCard.tsx` | component | request-response | `SIBLING:src/components/Projects.tsx` lines 41-85 | exact (card idiom) |
| `src/components/Poster.tsx` | component | request-response | `SIBLING:src/components/Projects.tsx` lines 49-56 | exact (lazy img in aspect box) |
| `src/components/Chip.tsx` (price/condition/filter chips) | component | request-response | `SIBLING:src/components/Projects.tsx` lines 74-83 | exact (tech-tag chip) |
| `src/components/CatalogBrowser.tsx` | component (`'use client'`) | event-driven (URL state) | none | no analog — use ARCHITECTURE Pattern 3 |
| `src/components/FilterBar.tsx` | component (`'use client'`) | event-driven | `SIBLING:src/components/Skills.tsx` lines 33-53 (grouped list w/ icon heading) | partial |
| `src/components/FilterSheet.tsx` (mobile bottom sheet) | component (`'use client'`) | event-driven | none | no analog |
| `src/components/GridSkeleton.tsx` | component | request-response | none | no analog |
| `src/components/EmptyState.tsx` | component | request-response | none | no analog |
| `src/lib/schema.ts` | model (zod) | transform | none | no analog — ARCHITECTURE § Proposed Schema |
| `src/lib/catalog.ts` | service (build-time accessor + projection) | transform | none | no analog — ARCHITECTURE Pattern 1 |
| `src/lib/filters.ts` | utility (pure) | transform | none | no analog |
| `src/lib/url-state.ts` / `useCatalogParams` hook | hook | event-driven | none | no analog — ARCHITECTURE Pattern 3 |
| `src/lib/base-path.ts` | utility | transform | none | no analog — ARCHITECTURE Pattern 4 |
| `src/lib/images.ts` | utility | transform | none | no analog — ARCHITECTURE Pattern 5 |
| `src/lib/format.ts` (price, condition labels) | utility | transform | none | no analog |
| `src/lib/__tests__/filters.test.ts`, `schema.test.ts` | test | — | none | no analog |
| `scripts/validate.ts` (+ `--write` deterministic writer) | script | file-I/O | none | no analog |
| `scripts/seed.ts` | script | file-I/O + request-response (TMDB fetch) | none | no analog |
| `data/catalog.json`, `data/seller.json`, `data/catalog.schema.json` | data | — | none | no analog |

## Pattern Assignments

### `next.config.ts` (config)

**Analog:** `SIBLING:next.config.ts` lines 1-11 (entire file)
```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "export",
  trailingSlash: true,
  images: {
    unoptimized: true,
  },
};

export default nextConfig;
```
**Phase 1 delta (D-28):** add `basePath` and `assetPrefix` derived from `process.env.NEXT_PUBLIC_BASE_PATH ?? ""` (strip trailing slash; pass `undefined` when empty). Keep all three sibling keys verbatim.

---

### `package.json` (config)

**Analog:** `SIBLING:package.json` lines 1-24
```json
{
  "name": "patrickclery-portfolio",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "lucide-react": "^0.577.0",
    "next": "^15.2.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@tailwindcss/postcss": "^4.0.0",
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "tailwindcss": "^4.0.0",
    "typescript": "^5"
  }
}
```
**Phase 1 delta (D-33/D-34):** name `dvd-seller`; bump to next 16.x / react 19.3 / lucide-react 1.x / TS ^5.9 / tailwind 4.3; add deps `zod`; devDeps `tsx`, `vitest`, `prettier`, `eslint`, `eslint-config-next`, `@types/node ^22`; scripts `prebuild: "npm run validate"`, `validate: "tsx scripts/validate.ts"`, `seed: "tsx scripts/seed.ts"`, `test: "vitest run"`, `lint`, `format`; `"engines": { "node": ">=22" }`. Drop `start` (static export has no server).

---

### `tsconfig.json` (config)

**Analog:** `SIBLING:tsconfig.json` lines 1-40 — copy verbatim. Key lines: `"strict": true`, `"moduleResolution": "bundler"`, `"resolveJsonModule": true` (needed for `catalog.json` import), `"paths": { "@/*": ["./src/*"] }`, `"plugins": [{ "name": "next" }]`, `include` has `next-env.d.ts`, `**/*.ts`, `**/*.tsx`, `.next/types/**/*.ts`.
**Phase 1 delta:** consider `"target": "ES2022"` (sibling uses ES2017) so `String.prototype.normalize` + `Array.prototype.at` type cleanly; add `"scripts/**/*.ts"` is already covered by `**/*.ts`.

---

### `postcss.config.mjs` (config)

**Analog:** `SIBLING:postcss.config.mjs` lines 1-6 — copy verbatim.
```javascript
const config = {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};
export default config;
```

---

### `.gitignore` (config)

**Analog:** `SIBLING:.gitignore` lines 86-113 (skip the JetBrains block, lines 1-84)
```gitignore
# dependencies
/node_modules
/.pnp
.pnp.js

# testing
/coverage

# next.js
/.next/
/out/

# production
/build

# misc
.DS_Store
*.pem

# debug
npm-debug.log*

# local env files
.env*.local

# typescript
*.tsbuildinfo
next-env.d.ts
```
**Phase 1 delta (D-34):** widen `.env*.local` to `.env*` (keep a committed `.env.example` via `!.env.example`); add `.cache/`, `photos/`, and image extensions outside `public/` (`*.jpg`, `*.jpeg`, `*.png`, `*.heic`, then `!public/**/*.png`). Do NOT ignore `.planning/` (sibling does; this repo commits it).

---

### `src/app/globals.css` (design tokens)

**Analog:** `SIBLING:src/app/globals.css` lines 1-43 — copy verbatim; these are the D-16 tokens.
```css
@import "tailwindcss";

@layer base {
  :root {
    --color-bg: #0F172A;
    --color-surface: #1E293B;
    --color-border: #334155;
    --color-text: #F8FAFC;
    --color-text-muted: #94A3B8;
    --color-accent: #22C55E;
    --color-accent-warm: #F59E0B;
  }

  html {
    scroll-behavior: smooth;
  }

  @media (prefers-reduced-motion: reduce) {
    html {
      scroll-behavior: auto;
    }
    *, *::before, *::after {
      animation-duration: 0.01ms !important;
      transition-duration: 0.01ms !important;
    }
  }

  body {
    background-color: var(--color-bg);
    color: var(--color-text);
  }

  ::selection {
    background-color: var(--color-accent);
    color: var(--color-bg);
  }

  *:focus-visible {
    outline: 2px solid var(--color-accent);
    outline-offset: 2px;
    border-radius: 4px;
  }
}
```
**Phase 1 delta:** optionally add an `@theme` block mapping the same vars to Tailwind color names (`--color-surface` etc. are already valid Tailwind v4 theme keys if moved into `@theme {}`), which would allow `bg-surface` instead of `bg-[var(--color-surface)]`. The sibling uses the arbitrary-value form everywhere; either is acceptable, but be consistent within this repo.

---

### `src/app/layout.tsx` (root layout / provider)

**Analog:** `SIBLING:src/app/layout.tsx` lines 1-45

**Imports + font setup** (lines 1-15) — copy verbatim:
```typescript
import type { Metadata } from "next";
import { Archivo, Space_Grotesk } from "next/font/google";
import "./globals.css";

const archivo = Archivo({
  subsets: ["latin"],
  variable: "--font-archivo",
  display: "swap",
});

const spaceGrotesk = Space_Grotesk({
  subsets: ["latin"],
  variable: "--font-space-grotesk",
  display: "swap",
});
```

**Metadata** (lines 17-28) — same shape, new copy:
```typescript
export const metadata: Metadata = {
  title: "Patrick Clery — Full-Stack Engineer",
  description: "...",
  openGraph: { title: "...", description: "...", url: "https://patrickclery.com", type: "website" },
};
```

**Body wiring** (lines 30-45):
```tsx
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html
      lang="en"
      className={`${archivo.variable} ${spaceGrotesk.variable}`}
    >
      <body className="font-[family-name:var(--font-space-grotesk)] antialiased">
        {children}
      </body>
    </html>
  );
}
```
**Phase 1 delta (D-32):** render `<Footer />` (TMDB attribution + logo) inside `<body>` after `{children}`. Headings everywhere use `font-[family-name:var(--font-archivo)]` (see Projects.tsx line 30). OG meta is Phase 2 — keep `metadata` minimal (title/description only).

---

### `src/app/page.tsx` (server route)

**Analog:** `SIBLING:src/app/page.tsx` lines 1-19
```tsx
import { Hero } from "../components/Hero";
import { Projects } from "../components/Projects";
// ...
export default function Home() {
  return (
    <main>
      <Hero />
      <Projects />
      {/* ... */}
      <Footer />
    </main>
  );
}
```
**Phase 1 delta (D-27, D-29):** server component; `import { getCards, getCounts } from "@/lib/catalog"` and `seller` from `@/lib/seller`; render `<Header counts blurb />` then `<Suspense fallback={<GridSkeleton />}><CatalogBrowser cards={getCards()} /></Suspense>`. Use the `@/` alias (sibling uses relative `../components/...`; D-35 says `@/*`). Never pass full `Movie[]`.

---

### `src/components/MovieCard.tsx` (component, exact analog)

**Analog:** `SIBLING:src/components/Projects.tsx`

**Imports** (line 1) — lucide named imports:
```typescript
import { ExternalLink, Terminal, BookOpen } from "lucide-react";
```

**Card shell — whole card is a link, border → accent on hover** (lines 41-47, 85):
```tsx
<a
  key={project.name}
  href={project.url}
  target="_blank"
  rel="noopener noreferrer"
  className="group cursor-pointer rounded-xl border border-[var(--color-border)] bg-[var(--color-surface)] overflow-hidden transition-colors duration-200 hover:border-[var(--color-accent)]"
>
  ...
</a>
```
Phase 1: replace `<a target=_blank>` with `next/link` `<Link href={`/movie/${slug}/`}>` (internal, basePath-aware). Keep the exact className string; poster box becomes `aspect-[2/3]`.

**Image in fixed-aspect box, lazy** (lines 48-57) — this is the Poster.tsx pattern:
```tsx
{project.preview && (
  <div className="relative w-full aspect-[16/9] overflow-hidden border-b border-[var(--color-border)]">
    <img
      src={project.preview}
      alt={`${project.name} preview`}
      className="w-full h-full object-cover object-top transition-transform duration-300 group-hover:scale-[1.02]"
      loading="lazy"
    />
  </div>
)}
```
Phase 1 delta (D-17/D-18/D-19): `aspect-[2/3]`, add `decoding="async"`, `width={342} height={513}`; `src` from `posterUrl(posterPath)`; when `posterPath` is null render placeholder `<div>` with title/year in Archivo + faint `<Film>` icon instead of `<img>`; when `status === "sold"` add `grayscale opacity-60` to img and overlay a rotated `SOLD` ribbon in `bg-[var(--color-accent-warm)]`; `reserved` → small warm pill top-right.

**Caption block + eyebrow label** (lines 58-73):
```tsx
<div className="p-6">
  <div className="flex items-start justify-between">
    <div className="flex items-center gap-3">
      <Icon className="w-5 h-5 text-[var(--color-accent)]" />
      <h3 className="font-[family-name:var(--font-archivo)] text-xl font-bold font-mono text-[var(--color-text)]">
        {project.name}
      </h3>
    </div>
  </div>
  <span className="mt-2 inline-block text-xs font-semibold uppercase tracking-wider text-[var(--color-accent-warm)]">
    {project.highlight}
  </span>
  <p className="mt-3 text-[var(--color-text-muted)] leading-relaxed">
    {project.description}
  </p>
```
Phase 1: shrink padding to `p-3`, title `text-sm font-semibold line-clamp-2`, "year · IMDb 8.2" line in `text-xs text-[var(--color-text-muted)]`. Drop the hover-only `ExternalLink` opacity trick (line 66) — D-17 forbids hover-only info.

**Chip / tag idiom** (lines 74-83) — reuse for price chip, condition chip, and filter chips (`Chip.tsx`):
```tsx
<div className="mt-4 flex flex-wrap gap-2">
  {project.tech.map((t) => (
    <span
      key={t}
      className="text-xs font-mono px-2 py-1 rounded bg-[var(--color-bg)] text-[var(--color-text-muted)] border border-[var(--color-border)]"
    >
      {t}
    </span>
  ))}
</div>
```
Phase 1: selected filter chips flip to `border-[var(--color-accent)] text-[var(--color-accent)]`; price chip uses `text-[var(--color-text)] font-semibold`; make filter chips `<button type="button" aria-pressed>` with `min-h-11 min-w-11` (44px, D-20).

**Component declaration convention** (line 26): `export function Projects() { ... }` — named export, PascalCase, no `React.FC`, data as a module-level `const` array. Prop types inline: `{ cards }: { cards: Card[] }`.

---

### `src/components/Header.tsx` (component)

**Analog:** `SIBLING:src/components/Projects.tsx` lines 28-36 (section + heading + muted paragraph)
```tsx
<section id="projects" className="px-6 py-24">
  <div className="mx-auto max-w-5xl">
    <h2 className="font-[family-name:var(--font-archivo)] text-3xl font-bold tracking-tight sm:text-4xl">
      Projects
    </h2>
    <p className="mt-4 text-[var(--color-text-muted)] max-w-2xl leading-relaxed">
      Open-source tools that bridge traditional engineering and AI-assisted
      development.
    </p>
```
Phase 1 delta (D-20/D-32): `max-w-7xl`, tighter `py-8`; `<h1>` site title; counts line "N available · M total" uses the `&middot;` separator idiom from `SIBLING:src/components/Footer.tsx` line 35; seller blurb in the muted `<p>`.

---

### `src/components/Footer.tsx` (TMDB attribution)

**Analog:** `SIBLING:src/components/Footer.tsx` lines 3-6, 34-38
```tsx
export function Footer() {
  return (
    <footer className="px-6 py-12 border-t border-[var(--color-border)]">
      <div className="mx-auto max-w-5xl text-center">
        ...
        <div className="mt-6 space-y-1 text-sm text-[var(--color-text-muted)]">
          <p>English (Native) &middot; French (Professional) &middot; Russian (Elementary)</p>
```
Phase 1: same shell; content = TMDB logo `<img src={withBasePath("/tmdb.svg")} alt="TMDB" loading="lazy">` + `<p>{catalog.attribution}</p>` (the literal "This product uses the TMDB API but is not endorsed or certified by TMDB."). Logo file goes in `public/`, so `src` MUST go through `withBasePath` (D-28).

---

### `src/components/FilterBar.tsx` (partial analog)

**Analog:** `SIBLING:src/components/Skills.tsx` lines 3-24 (grouped config array) and 33-53 (group heading with icon + list)
```tsx
const skillGroups = [
  { category: "Backend", icon: Server, skills: [...] },
  ...
];
...
<div key={group.category}>
  <div className="flex items-center gap-2 mb-4">
    <Icon className="w-4 h-4 text-[var(--color-accent-warm)]" />
    <h3 className="text-sm font-semibold uppercase tracking-wider text-[var(--color-accent-warm)]">
      {group.category}
    </h3>
  </div>
```
Phase 1: model each chip group (Genre / Decade / Rating / Availability) as one entry in a module-level config array with `label`, `icon`, `param key`, `options`; render the group heading with this exact eyebrow style; options rendered with `Chip.tsx`. Sort `<select>` and search `<input>` have no sibling analog — style them with `bg-[var(--color-surface)] border border-[var(--color-border)] rounded-lg px-3 min-h-11 text-sm` to match the chip palette. Desktop: `overflow-x-auto flex gap-2` chip rows; mobile: same groups inside `FilterSheet`.

---

### `public/.nojekyll`

**Analog:** `SIBLING:public/.nojekyll` — zero-byte file, committed. Required because `_next/` is underscore-prefixed.

---

## Shared Patterns

### Design tokens via CSS vars + Tailwind arbitrary values
**Source:** `SIBLING:src/app/globals.css` lines 4-12; usage in `SIBLING:src/components/Projects.tsx` line 46
**Apply to:** every component
Sibling never uses raw Tailwind colors for surfaces/text; always `bg-[var(--color-surface)]`, `border-[var(--color-border)]`, `text-[var(--color-text-muted)]`, `text-[var(--color-accent)]`, `text-[var(--color-accent-warm)]`. Only exception seen: `hover:text-green-400` in Footer.tsx line 10.

### Heading font
**Source:** `SIBLING:src/components/Projects.tsx` line 30, `Skills.tsx` line 30
**Apply to:** Header title, placeholder-poster title, sheet title
```
className="font-[family-name:var(--font-archivo)] text-3xl font-bold tracking-tight sm:text-4xl"
```
Body font is set once on `<body>` (`layout.tsx` line 40); components never set it.

### Container / section rhythm
**Source:** `SIBLING:src/components/Projects.tsx` lines 28-29, 37
**Apply to:** Header, grid wrapper, footer
`<section className="px-6 py-24"><div className="mx-auto max-w-5xl">` … `<div className="mt-12 grid gap-6 sm:grid-cols-2">`. Phase 1 uses `max-w-7xl` and `grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 xl:grid-cols-6 gap-3 sm:gap-4` (D-20).

### Icons
**Source:** `SIBLING:src/components/Projects.tsx` line 1, 61; `Skills.tsx` line 39
**Apply to:** all components
Named imports from `lucide-react`; sized `w-4 h-4` / `w-5 h-5`; colored with a token; stored as a component reference in config arrays (`icon: Terminal`) and rendered via `const Icon = group.icon;`. Phase 1 icons: `Film` (placeholder), `Search`, `SlidersHorizontal` (Filters button), `X` (clear/close), `ArrowUpDown` (sort).

### Images
**Source:** `SIBLING:src/components/Projects.tsx` lines 49-56
**Apply to:** Poster.tsx, Footer TMDB logo
Plain `<img loading="lazy">` in a fixed-aspect `relative overflow-hidden` box with `object-cover`; never `next/image`. Add `decoding="async"` + explicit `width`/`height` (Phase 1 addition for layout stability).

### Component declaration
**Source:** all sibling components
`export function Name()` — PascalCase file `Name.tsx`, one component per file, inline prop types, module-level config `const` arrays, 2-space indent, double quotes, trailing commas, semicolons. No default exports except Next route files (`layout.tsx`, `page.tsx`).

### Static-export config
**Source:** `SIBLING:next.config.ts`, `SIBLING:public/.nojekyll`, `SIBLING:.github/workflows/deploy.yml` (Phase 2 — not copied now)
`output: "export"` + `trailingSlash: true` + `images.unoptimized: true` + `.nojekyll`. Deploy workflow reference for Phase 2: `deploy.yml` lines 17-30 (`actions/checkout@v4` → `setup-node@v4` node `'20'` cache `'npm'` → `npm ci` → `npm run build` → `upload-pages-artifact@v3 path: out/`); Phase 2 bumps node to `'22'` and adds `env: NEXT_PUBLIC_BASE_PATH` on the build step.

## No Analog Found

Sibling has no data layer, client state, scripts, or tests. Planner should use `.planning/research/ARCHITECTURE.md` (already read; section refs below) and CONTEXT.md decisions directly.

| File | Role | Data Flow | Reason / Reference |
|------|------|-----------|--------------------|
| `src/lib/schema.ts` | model (zod) | transform | ARCHITECTURE § "Proposed `catalog.json` Entry Schema" lines 369-449, with D-02/D-03/D-10 tweaks (`condition` enum `new|like-new|very-good|good|acceptable`; `confirmedBy` adds `"seed"`); plus `Seller` schema (D-31) |
| `src/lib/catalog.ts` | service | transform | ARCHITECTURE Pattern 1 (lines 143-152): `CatalogSchema.parse(raw)` at module load, `getCards()` projection |
| `src/lib/filters.ts` | utility (pure) | transform | No analog; D-21..D-25 define semantics; must be React-free for Vitest |
| `src/lib/url-state.ts` / hook | hook | event-driven | ARCHITECTURE Pattern 3 (lines 193-205): `useSearchParams` + `router.replace(..., { scroll: false })`; D-26 key names; defaults omitted |
| `src/lib/base-path.ts` | utility | transform | ARCHITECTURE Pattern 4 (lines 218-229): `BASE_PATH` + `withBasePath(p)` |
| `src/lib/images.ts` | utility | transform | ARCHITECTURE Pattern 5 (lines 241-256): `posterUrl(path, size)`, `NEXT_PUBLIC_IMAGE_MODE` switch; D-18 says null → placeholder component, not SVG URL |
| `src/lib/format.ts` | utility | transform | D-05 `Intl.NumberFormat("en-CA", { style: "currency", currency })` with `.00` stripped; D-03 condition labels |
| `src/components/CatalogBrowser.tsx` | component (client) | event-driven | ARCHITECTURE Pattern 3; must sit inside `<Suspense>` in `page.tsx` |
| `src/components/FilterSheet.tsx` | component (client) | event-driven | No analog; Claude's discretion — native `<dialog>` recommended (no new UI lib) |
| `src/components/GridSkeleton.tsx`, `EmptyState.tsx` | component | request-response | No analog; use surface/border tokens, `aspect-[2/3]` boxes with `animate-pulse` |
| `scripts/validate.ts` | script | file-I/O | No analog; parse `data/catalog.json` + `data/seller.json` with zod, enforce slug/`(tmdbId, edition)` uniqueness, `--write` emits deterministic output (D-09), non-zero exit on failure; emit `data/catalog.schema.json` via `z.toJSONSchema` (zod 4) |
| `scripts/seed.ts` | script | file-I/O + fetch | No analog; `fetch` TMDB `/3/movie/{id}?append_to_response=credits,external_ids` with `Authorization: Bearer ${process.env.TMDB_API_TOKEN}` (load `.env.local` via `process.loadEnvFile`), write through the same writer as validate (D-15). Requires human checkpoint for the token (D-14) |
| `src/lib/__tests__/*.test.ts` | test | — | No analog (sibling has no tests); Vitest, `describe/it/expect` |
| `.nvmrc`, `.prettierrc`, `eslint.config.mjs`, `vitest.config.ts` | config | — | No analog; keep minimal (D-35): `eslint-config-next` core-web-vitals flat config, Prettier defaults |
| `data/catalog.json`, `data/seller.json`, `data/catalog.schema.json` | data | — | Generated by `seed.ts` / `validate.ts --write`; shape from D-09, D-31 |

## Metadata

**Analog search scope:** `/home/patrick/github.com/patrickclery/patrickclery.github.io` — `next.config.ts`, `package.json`, `tsconfig.json`, `postcss.config.mjs`, `.gitignore`, `src/app/{layout,page}.tsx`, `src/app/globals.css`, `src/components/{Projects,Skills,Footer}.tsx`, `.github/workflows/deploy.yml`, `public/.nojekyll`; plus `/home/patrick/github.com/patrickclery/dvd-seller/.planning/research/ARCHITECTURE.md` for no-analog patterns
**Files scanned:** 12 sibling files (all git-tracked, verified with `git ls-files`) + 1 research doc
**Pattern extraction date:** 2026-09-21
