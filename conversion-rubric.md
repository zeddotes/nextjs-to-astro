# Component conversion rubric

Choose the **smallest** client/runtime surface that meets the requirement. Default: `.astro` only.

---

## Decision tree

```
START: React/TSX component used by a page
│
├─ RSC async server component (no hooks, no "use client")?
│  └─ YES → .astro frontmatter fetch + template (default for App Router pages)
│
├─ Pure markup + props, no hooks, no browser APIs?
│  └─ YES → Convert to .astro (inline or import)
│
├─ Only needs form submit / fetch / DOM toggle without a framework?
│  └─ YES → .astro + <script> (or scoped script with type="module")
│
├─ Needs useState, useEffect, context, Radix, complex client lib?
│  └─ YES → Keep as framework component + client:* directive
│       ├─ SSR-safe, SEO needs initial HTML? → client:load or client:visible
│       └─ SSR causes mismatch or CSP inline-style issues? → client:only="react"
│
├─ Marked "use client" but only renders static children?
│  └─ YES → Collapse into parent .astro; delete wrapper
│
└─ CMS rich text / MDX with many custom types?
   ├─ Low effort to map types in Astro? → Astro components per block type
   └─ High effort? → Keep one React renderer (minimize scope)
```

**Dashboard / heavy client state:** flag in inventory; section may remain React-heavy — not forced to `.astro`.

---

## Target by pattern

| Pattern | Target | Notes |
|---------|--------|-------|
| RSC async page component | `.astro` | Data in frontmatter; no `"use client"` |
| `"use client"` leaf widget | React island | Deliberate `client:*`; smallest scope |
| Page shell, sections, grids | `.astro` | Compose in `src/pages/*.astro` |
| Typography, cards, static lists | `.astro` | Props via `Astro.props` |
| Header / footer / nav | `.astro` | Shared layout partials |
| Image with known dimensions | `.astro` + `astro:assets` `<Image />` | Avoid `next/image` |
| Image in React island | JSX `<img />` | Astro does not optimize island images |
| Link | `<a>` or thin `SafeLink` | No `next/link` |
| FAQ accordion (Radix) | React island | One island per page if possible |
| Contact email capture | `.astro` + `<script>` + `POST /api/...` | Shared Zod in `src/lib` |
| Modal with focus trap | React island or small script | Prefer island only if trap lib is React-only |
| Theme toggle | `<script>` or island | Depends on implementation |
| Analytics | `public/scripts/*.js` | Avoid inline when CSP strict |

---

## `client:*` directive guide

| Directive | When |
|-----------|------|
| `client:load` | Needed immediately on page load (accordion open, critical widget) |
| `client:idle` | Can wait until browser idle |
| `client:visible` | Below fold / when scrolled into view |
| `client:media="(min-width: ...)"` | Desktop-only interactivity |
| `client:only="react"` | No SSR for this component (CSP / hydration mismatch) |

**Anti-pattern:** `client:load` on every imported React component — bundles JS across the site.

---

## React wrapper anti-patterns

Remove wrappers that only:

- Pass props through to children
- Render a single child `.astro` could own
- Duplicate layout already in `Layout.astro`
- Exist because Next required a client boundary but add no interactivity

Example to collapse:

```tsx
// AboutPage.tsx — delete after migration
export function AboutPage({ dict }) {
  return <main>...</main>;
}
```

```astro
---
// about.astro — own the markup
import { getDictionary } from "../lib/content";
const dict = await getDictionary();
---
<main>...</main>
```

Run a dedicated cleanup pass after routes work: remove thin wrappers in a separate commit so diffs stay reviewable.

---

## `.astro` component checklist

- [ ] Data loaded in frontmatter (not in template as async)
- [ ] No `useState` / `useEffect` in `.astro` (use island or `<script>`)
- [ ] Styles: scoped `<style>`, CSS Modules, or existing global CSS import
- [ ] Props typed via interface + `Astro.props`
- [ ] No default client JS emitted
- [ ] Valid HTML (Astro 7 Rust compiler rejects unclosed tags)

---

## Client `<script>` checklist

Use when:

- Form `fetch` to API route
- Toggle classes / `aria-expanded`
- Small DOM queries

```astro
<form id="contact-form">...</form>
<script>
  const form = document.getElementById("contact-form");
  form?.addEventListener("submit", async (e) => {
    e.preventDefault();
    // fetch("/api/contact", { method: "POST", ... })
  });
</script>
```

- [ ] No secrets in client script
- [ ] Progressive enhancement where possible

---

## Island checklist

- [ ] Single responsibility (one widget per file if practical)
- [ ] Props serializable from `.astro` frontmatter
- [ ] `client:*` chosen deliberately (not default `load`)
- [ ] Dependencies in `package.json` `dependencies` if needed at SSR/build for manifest (host-specific; verify build)

---

## Verification per component

1. Page builds without importing `next/*`
2. Network tab: no unexpected large JS chunk on static pages
3. Interactive behavior works (keyboard, form errors)
4. Visual parity with Next version (spacing, responsive)
