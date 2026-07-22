---
name: nextjs-to-astro
description: "Migrate Next.js 13-16 projects to Astro 7. Handles route mapping, API transitions, middleware alignment, and converting React components to Astro-first architecture."
---

# Next.js → Astro 7 migration

Playbook for migrating Next.js 13–16 to **Astro 7**: plan scrutiny, structural move, Astro-first component conversion, CSP alignment, and minimal client islands.

**Self-contained** — planning scrutiny is built into Phase 0. No other Cursor skills required.

**Out of scope:** CSS/Tailwind/styling framework changes; hosting packaging (adapters, CDN rewrites, artifact size). Stop when `astro build` passes locally.

**Official docs:**

- [Migrating from Next.js](https://docs.astro.build/en/guides/migrate-to-astro/from-nextjs/)
- [Upgrade to Astro v7](https://docs.astro.build/en/guides/upgrade-to/v7/)

---

## Phase 0 — Plan scrutiny (mandatory)

Before writing migration code, follow [planning-rubric.md](./planning-rubric.md).

**Agent behavior:**

1. **Explore first** — `package.json`, `next.config.*`, `app/` / `pages/`, middleware, env usage, `CONTEXT.md` / ADRs
2. **Interview one decision at a time** — recommend a default; wait for confirmation when ambiguous
3. **Answer from code when possible** — grep/read instead of asking factual questions
4. **Challenge vague plans** — CMS, auth, SSR scope must be concrete before implementation
5. **Cross-check code vs plan** — surface contradictions (e.g. "all static" but `getServerSideProps` exists)
6. **Stress-test with scenarios** — forms + CSP, preview mode, i18n, protected routes
7. **Capture decisions inline** — update `CONTEXT.md` or `docs/migration.md` as terms settle
8. **ADRs only for irreversible choices** — hard to reverse, surprising without context, real trade-off

**Decisions table** (resolve before Phase 1):

| Decision | What to nail down |
|----------|-------------------|
| Next version | 13 / 14 / 15 / 16 (from `package.json`) — see [nextjs-versions.md](./nextjs-versions.md) |
| Router | Pages, App, or both |
| `output` | `static`, `server`, or hybrid; which routes are prerendered |
| React islands | Which components stay client-side; default Astro-first |
| Forms | API route + client script vs `astro:actions` |
| Content | CMS at build time, exported JSON, Content Collections, etc. |
| Admin / studio | Drop, separate app, or keep |
| Analytics / third-party JS | External files vs inline (CSP) |
| Env | `astro:env` schema in `astro.config` |
| Styling | Carry over existing CSS stack as-is (out of scope to change) |

**Gate:** Do not scaffold Astro or migrate routes until the inventory outline and decisions above are written down (chat summary + project doc).

---

## Phase 1 — Detect and audit

Build a **migration inventory** (route → Astro file, data source, prerender yes/no, Next feature flags).

### Detection

```bash
# Router and version
test -d app && echo "App Router"
test -d pages && echo "Pages Router"
node -p "require('./package.json').dependencies.next || require('./package.json').devDependencies.next"

# Next 13–16 patterns
rg '"use server"|"use client"' app src pages 2>/dev/null
rg 'unstable_cache|revalidatePath|revalidateTag|draftMode' .
rg 'generateStaticParams|getStaticProps|getServerSideProps' .
rg "runtime = 'edge'" app pages src 2>/dev/null
```

### Grep / scan checklist

```bash
# Config & packages
rg -l 'from ["\']next|next/' --glob '!node_modules'
rg 'next\.config|next-env\.d\.ts' .
rg '"next"|eslint-config-next' package.json

# Routers
find app pages -maxdepth 3 2>/dev/null

# APIs
rg 'route\.ts|pages/api' app pages 2>/dev/null

# Next-specific APIs
rg 'next/image|next/link|next/font|next/head' .
```

### Inventory columns

| Next path | Astro target | Data at build / request | `prerender` | Next feature |
|-----------|--------------|-------------------------|-------------|--------------|
| … | `src/pages/...` | … | true / false | RSC, Server Action, edge, etc. |

Flag for removal (do not migrate): legacy static-export scripts, unused CMS studio in the app bundle, dead deploy scripts tied to Next.

---

## Phase 2 — Scaffold Astro 7

**Requirements:** Node 22+, `astro@^7`.

1. Add `astro@^7`, integrations, and framework packages (`@astrojs/react` only if islands remain; `@astrojs/mdx` if MDX).
2. Run `npx @astrojs/upgrade` when upgrading an existing Astro scaffold.
3. Create `astro.config.mjs` / `.ts`:
   - `srcDir: "./src"` (or match repo convention)
   - `site` URL for sitemaps/canonicals
   - `integrations: [react()]` if needed
   - `env.schema` with `envField` — prefer `astro:env/client` and `astro:env/server` imports
   - Adapter line only if user chose hosting in Phase 0; otherwise omit or comment
4. **`tsconfig.json`:**

```json
{
  "extends": "astro/tsconfigs/base",
  "include": [".astro/types.d.ts", "**/*"],
  "exclude": ["dist"]
}
```

5. Create `src/layouts/Layout.astro`, `src/pages/`, `src/env.d.ts` if custom types needed.
6. Wire existing global CSS import in layout — **do not change** the CSS stack.

Do **not** remove `next` from `package.json` until Phase 6 — both may coexist briefly during migration.

---

## Phase 3 — Routes and data

See [mapping.md](./mapping.md) for the full route/API table.

### Patterns

- **App Router** `app/foo/page.tsx` → `src/pages/foo.astro`
- **Route groups** `(marketing)/about/page.tsx` → URL `/about`, file `src/pages/about.astro`
- **Dynamic** `[slug]` → `[slug].astro`; use `Astro.params` in frontmatter
- **Layouts** nested `layout.tsx` → `Layout.astro` + composition or nested layout components
- **Metadata** `export const metadata` / `generateMetadata` → props passed to `Layout` + `<title>` / meta tags
- **Data** move fetches to `.astro` frontmatter or `src/lib/*`; prefer build-time JSON over runtime CMS for marketing sites
- **Prerender** default static; `export const prerender = false` only for SSR-only routes
- **`import.meta.glob()`** for file batches — **`Astro.glob()` removed in Astro 6+**

### Astro 7 notes

- **Markdown:** Sätteri is default; remark/rehype plugins → `@astrojs/markdown-remark`
- **HTML:** Rust compiler is strict — fix unclosed tags during conversion
- **Reserved:** avoid naming unrelated files `src/fetch.ts` (advanced routing entry)

### Content export (optional)

If moving off live CMS fetches at runtime:

1. Export content to `content/**/*.json` (or Content Collections).
2. Add export script under `scripts/` (e.g. `scripts/export-cms-content.mjs`).
3. Implement `src/lib/content.ts` to load and type content at build time.
4. Document vocabulary in `CONTEXT.md` (slug rules, rich-text format, listing order).

---

## Phase 4 — Component conversion

Follow [conversion-rubric.md](./conversion-rubric.md).

Summary:

1. Static chrome → `.astro`
2. CMS rich text / MDX custom types → Astro components per block, or one React renderer if conversion cost is high
3. Images → `astro:assets` `<Image />` in `.astro`; plain `<img>` in React islands — no `next/image`
4. Links → `<a>` or shared helper — no `next/link`
5. Interactivity → smallest island + `client:load` or `client:only="react"` (CSP / hydration mismatch)
6. Forms → Astro markup + `<script>` posting to `src/pages/api/*.ts`

Collapse thin React page wrappers (`AboutPage.tsx` that only passes props) into `.astro` during cleanup.

Dashboard-heavy / high client-state sections: flag in inventory; may stay React-heavy — not forced to `.astro`.

---

## Phase 5 — API and server logic

| Next | Astro |
|------|-------|
| `app/api/foo/route.ts` | `src/pages/api/foo.ts` — export `GET` / `POST` as `APIRoute` |
| `pages/api/foo.ts` | same |
| Server Actions / `"use server"` | `APIRoute` + client `fetch` unless Phase 0 chose `astro:actions` |

Example handler:

```ts
import type { APIRoute } from "astro";

export const POST: APIRoute = async ({ request }) => {
  const body = await request.json();
  return new Response(JSON.stringify({ status: "ok" }), {
    status: 200,
    headers: { "Content-Type": "application/json" },
  });
};
```

Shared validation/logic → `src/lib/*.ts` (Zod, email senders, etc.).

**Middleware:** Next `middleware.ts` → `src/middleware.ts` (`defineMiddleware` from `astro:middleware`). CSP / security headers → `astro.config` `security.csp` (`directives`, `scriptDirective` / `styleDirective` with `resources`). Move inline analytics to `public/scripts/*.js` when CSP is strict.

Delete `src/actions/index.ts` if the project standard is API routes + client scripts.

---

## Phase 6 — Config parity and prune

### Config parity

- **Redirects / headers** from `next.config.js` → `astro.config` `redirects` or hosting config (document only)
- **Images domains** from `next.config` `images.remotePatterns` → not needed if using `astro:assets` or known URLs
- **TypeScript:** update paths/aliases; remove `next-env.d.ts`
- **CMS studio:** remove in-app studio if content is file-based at build time

### Delete when build passes

- `app/`, `pages/` (Next)
- `next.config.js`, `next-env.d.ts`
- `next`, `eslint-config-next` from `package.json`
- Unused CMS runtime client, legacy metadata helpers, Next test harness
- Dead React page wrappers superseded by `.astro`

### Verify

```bash
nvm use  # if applicable
node -v   # expect 22+
pnpm run build   # or npm/yarn per repo

rg 'from ["\']next|next/' --glob '!node_modules'   # expect no matches
rg 'Astro\.glob' .          # expect no matches
rg 'client:(load|only|idle|visible)' src components  # only intentional islands
astro preview  # smoke-test key routes
```

- `CONTEXT.md` / `docs/migration.md` matches final content layout
- Contact/API routes respond locally

### Stuck?

Use the repo's git history if migration was started previously:

```bash
git log --oneline --grep='astro\|migrate'
git log --oneline -- astro.config.mjs src/pages
git show <commit> --stat
```

---

## Hosting note

Deploy adapters, CDN rewrites, and artifact packaging are **not** covered here. After local `astro build` succeeds, use the host's Astro adapter docs.

---

## Appendices

- [planning-rubric.md](./planning-rubric.md) — Phase 0 plan scrutiny
- [mapping.md](./mapping.md) — routes, APIs, env, middleware
- [conversion-rubric.md](./conversion-rubric.md) — component target decision tree
- [nextjs-versions.md](./nextjs-versions.md) — Next 13–16 version-specific flags
