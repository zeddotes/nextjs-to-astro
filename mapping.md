# Next.js → Astro mapping reference

Quick lookup during migration to **Astro 7**. Workflow: [SKILL.md](./SKILL.md). Version flags: [nextjs-versions.md](./nextjs-versions.md).

---

## File-based routing

| Next.js (Pages Router) | Astro |
|------------------------|-------|
| `pages/index.tsx` | `src/pages/index.astro` |
| `pages/about.tsx` | `src/pages/about.astro` |
| `pages/blog/index.tsx` | `src/pages/blog/index.astro` |
| `pages/blog/[slug].tsx` | `src/pages/blog/[slug].astro` |
| `pages/blog/tag/[tag].tsx` | `src/pages/blog/tag/[tag].astro` |
| `pages/404.tsx` | `src/pages/404.astro` |
| `pages/_app.tsx` | `src/layouts/Layout.astro` (global shell) |
| `pages/_document.tsx` | `<html>` in `Layout.astro` |

| Next.js (App Router) | Astro |
|----------------------|-------|
| `app/page.tsx` | `src/pages/index.astro` |
| `app/about/page.tsx` | `src/pages/about.astro` |
| `app/(group)/foo/page.tsx` | `src/pages/foo.astro` (group not in URL) |
| `app/blog/[slug]/page.tsx` | `src/pages/blog/[slug].astro` |
| `app/layout.tsx` | `src/layouts/Layout.astro` |
| `app/blog/layout.tsx` | nested layout component or partial in `src/layouts/` |
| `app/loading.tsx` | no direct equivalent; skeleton in page or client island |
| `app/error.tsx` | `src/pages/500.astro` or error handling in layout |
| `app/not-found.tsx` | `src/pages/404.astro` |

**Dynamic params:** `params.slug` (Pages) / awaited `params` (App 15+) → `Astro.params.slug` in frontmatter.

**Search params:** `searchParams` (App) → `Astro.url.searchParams` (SSR) or client `URLSearchParams`.

---

## App Router-only patterns

| Next.js (App Router) | Astro |
|----------------------|-------|
| `template.tsx` | remount via client island or `<script>` if needed; usually omit |
| `default.tsx` | parallel route slot — compose as separate pages or partials |
| `@modal/(.)photo/[id]` intercepting | client island + history API, or dedicated modal route |
| `generateMetadata()` | compute in frontmatter; pass to layout |
| `opengraph-image.tsx` | `src/pages/og/[slug].png.ts` `APIRoute`, static asset, or build-time image |
| `icon.tsx` / `apple-icon.tsx` | `public/favicon.ico` or dynamic `APIRoute` |
| `sitemap.ts` / `robots.ts` | see Metadata & SEO below |

---

## Data fetching

| Next.js | Astro |
|---------|-------|
| `getStaticProps` | fetch/load in `.astro` frontmatter at build |
| `getStaticPaths` | `getStaticPaths()` export on dynamic route (static/hybrid) or `getCollection` |
| `getServerSideProps` | frontmatter on SSR route (`prerender = false`) |
| `generateStaticParams` (App) | `getStaticPaths()` or prerendered dynamic routes |
| RSC `async function Page()` fetch | frontmatter in `.astro` |
| `fetch(..., { next: { revalidate } })` | build-time data or explicit SSR fetch; no Next cache API |
| `unstable_cache` | build-time export or SSR refetch; external cache if needed |
| `revalidatePath` / `revalidateTag` | rebuild or CMS export script; no runtime invalidation in static mode |
| `use client` + client fetch | `<script>` or React island |
| `Astro.glob()` (legacy Astro) | **`import.meta.glob()`** — removed in Astro 6+ |

Prefer **build-time** JSON/files for marketing content; document loading in `src/lib/content.ts`.

---

## Caching (Next 14–16)

| Next.js | Astro 7 |
|---------|---------|
| `fetch` with default cache (15+) | explicit build-time or SSR fetch in frontmatter |
| `export const revalidate = N` | static rebuild cadence or CDN `routeRules` (host-specific) |
| `export const dynamic = 'force-dynamic'` | `export const prerender = false` |
| `draftMode()` | custom preview cookie + middleware; or separate preview deploy |

---

## API routes

| Next.js | Astro |
|---------|-------|
| `pages/api/contact.ts` | `src/pages/api/contact.ts` |
| `app/api/contact/route.ts` | `src/pages/api/contact.ts` |
| `export default function handler(req, res)` | `export const GET/POST: APIRoute = async ({ request }) => new Response(...)` |
| `NextRequest` / `NextResponse` | standard `Request` / `Response` |
| `req.body` (parsed) | `await request.json()` |
| `export const runtime = 'edge'` | adapter-specific; document in inventory — no universal 1:1 |
| Route segment config | Astro route `prerender = false` for SSR APIs |

**Server Actions** (`"use server"`) → `APIRoute` + `fetch` from client, or `astro:actions` if chosen in Phase 0 scrutiny.

---

## Metadata & SEO

| Next.js | Astro |
|---------|-------|
| `export const metadata = { title }` (App) | props to `Layout` + `<title>` |
| `generateMetadata()` | compute in frontmatter, pass to layout |
| `<Head>` / `next/head` (Pages) | `<head>` in `Layout.astro` or page |
| `app/sitemap.ts` | `src/pages/sitemap.xml.ts` (`export const GET: APIRoute`) or `@astrojs/sitemap` |
| `app/robots.ts` | `public/robots.txt` or `src/pages/robots.txt.ts` (`APIRoute`) |
| `next-seo` package | manual meta in layout or small helper |

---

## Assets & styling

| Next.js | Astro |
|---------|-------|
| `next/image` | `astro:assets` `<Image />` in `.astro`; `<img>` in React islands |
| `next/link` | `<a href="...">` |
| `next/font` | `@fontsource/*` or `<link>` in layout |
| CSS Modules `*.module.css` | same in `.astro` / components |
| Global CSS in `_app` | import in `Layout.astro` |
| `next/script` | `<script src="...">` in layout or `public/` |
| styled-components / CSS-in-JS | carry over in islands; may need `client:only="react"` under strict CSP |

**Styling migration is out of scope** — CSS Modules, global CSS, Tailwind, styled-components: wire existing imports; do not upgrade CSS frameworks during this migration.

### `next/image` → `astro:assets` example

```astro
---
import { Image } from "astro:assets";
import hero from "../assets/hero.png";
---
<Image src={hero} alt="Hero" width={1200} height={630} />
```

---

## Environment variables

| Next.js | Astro |
|---------|-------|
| `process.env.SECRET` (server) | `astro:env/server` with `env.schema` (`context: "server"`, `access: "secret"`) |
| `process.env.NEXT_PUBLIC_*` | `PUBLIC_*` via `astro:env/client` (`access: "public"`) |
| `.env.local` | `.env` (Astro loads per docs) |
| `NEXT_PUBLIC_SITE_URL` | `PUBLIC_SITE_URL` or `site` in `astro.config` |

Define schema in `astro.config`:

```js
import { defineConfig, envField } from "astro/config";

export default defineConfig({
  env: {
    schema: {
      API_KEY: envField.string({ context: "server", access: "secret" }),
      PUBLIC_ANALYTICS_ID: envField.string({ context: "client", access: "public" }),
    },
  },
});
```

Usage:

```astro
---
import { PUBLIC_ANALYTICS_ID } from "astro:env/client";
import { API_KEY } from "astro:env/server";
---
```

---

## Middleware

| Next.js | Astro |
|---------|-------|
| `middleware.ts` at root | `src/middleware.ts` (`defineMiddleware` from `astro:middleware`) |
| CSP / security headers in middleware | `astro.config` `security.csp` (preferred for static CSP) |
| Redirects in middleware | `astro.config` `redirects` or middleware `redirect()` |
| Locale prefix routing | Astro `i18n` config or `[lang]/` dynamic segments |

---

## Config file

| Next.js | Astro |
|---------|-------|
| `next.config.js` | `astro.config.mjs` |
| `images`, `rewrites`, `redirects`, `headers` | partial parity in `astro.config`; rest on host |
| `experimental` flags | Astro integrations / `vite` config |
| `transpilePackages` | `vite.ssr.noExternal` / `optimizeDeps` |

---

## Astro 7 specifics

| Topic | Notes |
|-------|-------|
| Node | 22+ required |
| Compiler | Rust-based; strict HTML — unclosed tags error |
| `Astro.glob()` | removed — use `import.meta.glob()` |
| Markdown | Sätteri default; `@astrojs/markdown-remark` for remark/rehype |
| `src/fetch.ts` | reserved for advanced routing — do not use for unrelated code |
| Endpoints with extension | no trailing slash (`/sitemap.xml` not `/sitemap.xml/`) |
| TypeScript | `.astro/types.d.ts`; `include` in `tsconfig.json` |

See [nextjs-versions.md](./nextjs-versions.md) and [Upgrade to Astro v7](https://docs.astro.build/en/guides/upgrade-to/v7/).

---

## Packages to remove (Phase 6)

- `next`, `eslint-config-next`
- `@next/font` (if present)
- CMS client used only at runtime after export to JSON

---

## Packages commonly added

- `astro@^7`, `@astrojs/react` (if islands), `@astrojs/mdx` (if MDX)
- `@astrojs/markdown-remark` (if remark/rehype plugins)
- Validation (`zod`), email SDKs, etc. moved to `src/lib`
