# Migration plan scrutiny

Mandatory **Phase 0** before writing migration code. Full workflow summary in [SKILL.md](./SKILL.md).

Goal: reach a shared, concrete migration plan — not a hand-wavy "move to Astro" checklist.

---

## Scrutiny rules

1. **Explore first** — read `package.json`, `next.config.*`, `app/` / `pages/`, `middleware.ts`, env files, and any `CONTEXT.md` / ADRs before asking questions.
2. **One decision at a time** — walk the decision tree below; provide a recommended default; wait for user confirmation when ambiguous.
3. **Answer from code when possible** — router type, Next version, SSR routes, Server Actions: grep/read the repo instead of asking.
4. **Challenge vague plans** — "move CMS to build time" is incomplete until you know: which CMS, export format, slug rules, refresh cadence, failure mode.
5. **Cross-check code vs plan** — if the user says "all static" but `getServerSideProps`, `export const dynamic = 'force-dynamic'`, or `prerender = false` patterns exist, surface the contradiction.
6. **Stress-test with scenarios** — use [Scenario probes](#scenario-probes) before implementation.
7. **Capture decisions inline** — update `CONTEXT.md` or create `docs/migration.md` as terms settle; do not batch at the end.
8. **ADRs sparingly** — offer an ADR only when all three hold: hard to reverse, surprising without context, result of a real trade-off.

---

## Decision tree (order)

```
1. Next version + router(s)
2. Astro output mode (static / server / hybrid) + per-route prerender
3. Data sources (build-time files, CMS export, live SSR fetch)
4. Interactivity inventory (islands vs .astro + <script>)
5. Forms (APIRoute + fetch vs astro:actions)
6. Auth / middleware / protected routes
7. Env secrets + public vars (astro:env schema)
8. Content export / Content Collections (if applicable)
9. What to delete (studio, dead scripts, Next-only harness)
10. Styling — confirm carry-over as-is (out of scope to change)
```

---

## Migration question bank

For each question: state what you found in code, recommend a default, ask only if the default is wrong for this project.

### 1. Router and version

- **Q:** Pages Router, App Router, or both?
- **How to answer:** `test -d app`, `test -d pages`; read `package.json` `dependencies.next`.
- **Default:** Migrate App Router `app/**/page.tsx` → `src/pages/*.astro`; Pages Router `pages/*` → same targets. Collide on URL? Resolve before Phase 2.

### 2. Output mode

- **Q:** Full static, full SSR (`output: 'server'`), or hybrid (mostly static + a few SSR/API routes)?
- **How to answer:** grep `getServerSideProps`, `export const dynamic`, `export const revalidate = 0`, API routes that must run at request time.
- **Default:** Marketing/content sites → `output: 'static'` with `export const prerender = false` only where inventory requires SSR.

### 3. React islands

- **Q:** Which components must stay React client-side?
- **How to answer:** grep `"use client"`, `useState`, `useEffect`, Radix/shadcn interactive widgets.
- **Default:** Astro-first; islands only for hooks, complex client libs, or focus-trap modals. See [conversion-rubric.md](./conversion-rubric.md).

### 4. Server Actions

- **Q:** How to replace `"use server"` handlers?
- **How to answer:** grep `"use server"` in `app/`, `src/actions`.
- **Default:** `src/pages/api/*.ts` (`APIRoute`) + client `fetch` or form `<script>`. Use `astro:actions` only if user explicitly chose it in Phase 0.

### 5. Content / CMS

- **Q:** Build-time content or live runtime fetches?
- **How to answer:** grep CMS client SDKs, `draftMode`, preview routes, `revalidateTag`.
- **Default:** Marketing sites → export to JSON or Content Collections at build; drop in-app studio unless user needs it.

### 6. Forms and contact

- **Q:** Contact/newsletter — API route + progressive enhancement?
- **Scenario:** Strict CSP — inline script blocked? → external `public/scripts/*.js` or `security.csp.scriptDirective.resources`.
- **Default:** `.astro` markup + `<script>` posting to `POST /api/contact`.

### 7. Middleware and auth

- **Q:** What does Next `middleware.ts` do — auth, redirects, locale, headers?
- **How to answer:** read `middleware.ts` matcher and branches.
- **Default:** Astro `src/middleware.ts` for auth/locals; CSP → `astro.config` `security.csp`; redirects → `astro.config` `redirects`.

### 8. Environment variables

- **Q:** Map `NEXT_PUBLIC_*` and server secrets to `astro:env` schema?
- **How to answer:** grep `process.env`, `.env*` files (names only — never commit secrets).
- **Default:** `env.schema` in `astro.config`; import from `astro:env/client` and `astro:env/server`.

### 9. Images

- **Q:** Replace `next/image` with `astro:assets` or plain `<img>`?
- **How to answer:** grep `next/image`, remote patterns in `next.config`.
- **Default:** `astro:assets` `<Image />` in `.astro` files for local/optimized images; plain `<img>` in React islands.

### 10. MDX / Markdown

- **Q:** MDX with remark/rehype plugins?
- **How to answer:** read `astro.config` / Next MDX config if present; grep remark plugins.
- **Default:** Astro 7 uses Sätteri by default. Existing remark/rehype → install `@astrojs/markdown-remark`.

### 11. i18n

- **Q:** Locale prefix routing in middleware?
- **Default:** Astro `i18n` config or manual `[lang]/` dynamic segments — document chosen approach.

### 12. Edge runtime routes

- **Q:** Any `export const runtime = 'edge'`?
- **Default:** Flag in inventory; no 1:1 in Astro — requires adapter-specific SSR; may stay as Node SSR route.

### 13. Dynamic OG / icons

- **Q:** `opengraph-image.tsx`, `icon.tsx`, `twitter-image.tsx`?
- **Default:** `src/pages/og/[slug].png.ts` endpoint, static assets, or build-time generation — pick per route in inventory.

### 14. Analytics and third-party JS

- **Q:** Inline vs `public/scripts/*.js`?
- **Default:** External files when CSP enabled.

### 15. Styling

- **Q:** Confirm existing CSS stack carries over unchanged?
- **Default:** Yes — CSS Modules, global CSS, styled-components, existing Tailwind setup: wire imports in `Layout.astro`; do not upgrade or migrate CSS frameworks in this skill.

### 16. What to delete

- **Q:** CMS studio, Next deploy scripts, Cypress Next harness, unused metadata helpers?
- **Default:** Remove after `astro build` passes; list in inventory "do not migrate" column.

---

## Code cross-check checklist

Grep patterns that often contradict a stated plan:

| Stated plan | Contradiction to grep |
|-------------|----------------------|
| "All pages static" | `getServerSideProps`, `dynamic = 'force-dynamic'`, `revalidate = 0`, `cookies()` in page |
| "No server logic" | `"use server"`, `pages/api`, `app/api/**/route.ts` |
| "No client JS" | Widespread `"use client"` without island plan |
| "Build-time content only" | Runtime CMS client in page components |
| "Simple images" | `next/image` with `fill`, remote `remotePatterns`, on-the-fly OG |
| "Edge APIs" | `runtime = 'edge'` without adapter choice |
| "Strict CSP" | Inline `<script>`, styled-components in islands without `client:only` plan |

Surface mismatches explicitly: "You said X, but the code shows Y — which wins?"

---

## Scenario probes

Walk through these before Phase 2. Invent project-specific variants when relevant.

1. **Contact form + CSP** — User submits form on `/contact`. Which script handles submit? Which API route validates and sends email? What appears if JS is disabled?

2. **Draft / preview** — Does Next `draftMode()` or preview API exist? How will editors preview unpublished content after migration?

3. **Protected `/admin`** — Middleware checks cookie. Where does auth run in Astro — middleware, API route, or host-level? What happens on 401?

4. **i18n `/en/about` vs `/fr/about`** — Same content, two locales. File structure? Default locale redirect?

5. **Dynamic blog `[slug]`** — 500 posts. `generateStaticParams` today? Keep full SSG or paginate? Slug collision rules?

6. **MDX with custom remark plugin** — Astro 7 Sätteri default. Install `@astrojs/markdown-remark` or rewrite plugin usage?

---

## Documentation capture

### CONTEXT.md / docs/migration.md

Update when a term or rule is resolved:

```markdown
## Migration decisions (Next → Astro 7)

- **Output:** static; SSR only `/api/*` and `/search`
- **Content:** Sanity exported to `content/**/*.json` at build via `scripts/export-cms.mjs`
- **Forms:** APIRoute + client script (not astro:actions)
- **Islands:** `FAQAccordion`, `CommandPalette` only
```

Include: slug rules, content vocabulary, listing order, env var renames (`NEXT_PUBLIC_*` → `PUBLIC_*`).

Do not couple to implementation details only experts care about — capture decisions a future maintainer needs.

### Minimal ADR (when warranted)

```markdown
# ADR-NNN: [Short title]

## Context
[Why this decision came up during Next → Astro migration]

## Decision
[What we chose]

## Consequences
[Trade-offs, what we gave up, what to watch]
```

Example triggers: drop in-app CMS studio, choose hybrid SSR over full static, keep dashboard section as React SPA shell.

---

## Exit criteria

Proceed to **Phase 1** only when all are true:

- [ ] Migration inventory **outline** exists (routes, APIs, layouts — even if rows are TBD)
- [ ] Decision table in [SKILL.md](./SKILL.md) Phase 0 is filled in
- [ ] Contradictions between plan and code are resolved or explicitly accepted
- [ ] User acknowledged trade-offs (island count, SSR scope, CMS export cadence)
- [ ] Decisions written to chat summary + `CONTEXT.md` or `docs/migration.md`
