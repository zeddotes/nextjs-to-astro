# Next.js version reference (13–16)

Quick lookup for version-specific APIs during migration to **Astro 7**. Workflow: [SKILL.md](./SKILL.md). Mapping: [mapping.md](./mapping.md).

Detect version from `package.json`:

```bash
node -p "require('./package.json').dependencies.next || require('./package.json').devDependencies.next"
```

---

## Summary table

| Version | Notable APIs | Astro 7 approach | Inventory flag |
|---------|--------------|------------------|----------------|
| **13** | App Router, `"use client"`, RSC pages, `next/font`, `generateMetadata` | `.astro` frontmatter for server data; islands for `"use client"` leaves; `@fontsource/*` or `<link>` for fonts | `APP_ROUTER`, `RSC`, `USE_CLIENT` |
| **14** | Server Actions stable, `next/og` (`ImageResponse`), PPR experimental | Server Actions → `APIRoute` + fetch; OG routes → endpoint or static assets | `SERVER_ACTIONS`, `OG_IMAGE` |
| **15** | Async `cookies()`, `headers()`, `params`, `searchParams`; caching default changes | Move async access to `.astro` frontmatter or middleware; no Next cache | `ASYNC_REQUEST_APIS`, `NEXT_CACHE` |
| **16** | Audit at migration time — check [Next.js docs](https://nextjs.org/docs) for new routing/cache APIs | Map each new API to build-time, SSR frontmatter, or API route | `AUDIT_DOCS` |

---

## Next.js 13

### App Router introduction

- `app/**/page.tsx` — default Server Components; no hooks in server components
- **Astro:** page logic in `.astro` frontmatter; client hooks only in islands

### `"use client"`

- Marks client boundary in App Router
- **Astro:** React island with deliberate `client:*` — see [conversion-rubric.md](./conversion-rubric.md)

### `next/font`

- Self-hosted or Google fonts via `next/font/google`
- **Astro:** `@fontsource/*` packages or `<link rel="stylesheet">` in `Layout.astro`

### `generateMetadata` / `export const metadata`

- **Astro:** compute in frontmatter; pass props to layout for `<title>` and meta tags

---

## Next.js 14

### Server Actions

- `"use server"` in files or inline on functions
- **Astro:** default `src/pages/api/*.ts` + client `fetch`; `astro:actions` only if chosen in Phase 0

### `next/og` (Open Graph)

- `ImageResponse` in `opengraph-image.tsx`
- **Astro:** `src/pages/og/[...].ts` `APIRoute` returning PNG, or pre-generated static images at build

### Partial Prerendering (experimental)

- Mix static shell + dynamic holes
- **Astro:** static `.astro` page + `server:defer` server islands or client islands for dynamic regions — document per route

---

## Next.js 15

### Async request APIs

- `cookies()`, `headers()`, `params`, `searchParams` are async in App Router
- **Astro:** `Astro.cookies`, `Astro.request.headers`, `Astro.params`, `Astro.url.searchParams` in frontmatter (sync in `.astro`; adapter context in middleware)

### Caching changes

- `fetch(..., { next: { revalidate: N } })`, default fetch caching shifted
- **Astro:** no Next cache layer — build-time fetch in frontmatter, or SSR fetch on `prerender = false` routes; use `routeRules` / CDN cache if needed (host-specific)

### `unstable_cache` / `revalidatePath` / `revalidateTag`

- **Astro:** build-time data refresh via rebuild or CMS export script; SSR routes refetch on each request unless external cache

---

## Next.js 16

At migration time:

1. Read `package.json` Next version and [Next.js release notes](https://github.com/vercel/next.js/releases)
2. Grep for APIs not listed above
3. Add rows to the project migration inventory with Astro equivalents
4. Do not assume 1:1 parity for bleeding-edge Next features — default to simplest Astro pattern that meets the requirement

---

## Pages Router (all versions)

Still common in Next 13–16 codebases:

| API | Astro 7 |
|-----|---------|
| `getStaticProps` | frontmatter fetch at build |
| `getStaticPaths` | `export function getStaticPaths()` on dynamic `.astro` route |
| `getServerSideProps` | frontmatter on route with `export const prerender = false` |
| `pages/_app.tsx` | `src/layouts/Layout.astro` |
| `pages/_document.tsx` | `<html>` shell in `Layout.astro` |
| `pages/api/*` | `src/pages/api/*` |

---

## Astro 7 requirements (all migrations)

| Requirement | Notes |
|-------------|-------|
| Node 22+ | Required |
| `astro@^7` | `npx @astrojs/upgrade` when upgrading existing Astro |
| No `Astro.glob()` | Use `import.meta.glob()` — removed in Astro 6+ |
| Strict HTML | Rust compiler errors on unclosed tags; fix during conversion |
| Sätteri Markdown | Default pipeline; remark/rehype → `@astrojs/markdown-remark` |
| Reserved `src/fetch.ts` | Advanced routing entry — avoid naming conflicts |
| `astro:env` | Prefer over raw `process.env` for typed secrets |

Official: [Upgrade to Astro v7](https://docs.astro.build/en/guides/upgrade-to/v7/)
