# nextjs-to-astro

Agent skill for migrating **Next.js 13–16** (Pages Router or App Router) to **Astro 7**.

Self-contained — no other skills required. Indexed on [skills.sh](https://skills.sh) when installed from a public GitHub repo.

## Install

### skills.sh / Skills CLI (recommended)

After this repo is public on GitHub:

```bash
# GitHub shorthand (repo root contains SKILL.md)
npx skills add zeddotes/nextjs-to-astro

# Full URL
npx skills add https://github.com/zeddotes/nextjs-to-astro

# Install into Cursor specifically
npx skills add zeddotes/nextjs-to-astro -a cursor
```

Browse and search skills: [skills.sh](https://skills.sh)

### Cursor (manual)

```bash
# Project-scoped
mkdir -p .cursor/skills
cp -r /path/to/nextjs-to-astro .cursor/skills/

# User-scoped (all projects)
cp -r /path/to/nextjs-to-astro ~/.cursor/skills/
```

### Local development

```bash
git clone <your-repo-url>
cd nextjs-to-astro
npx skills add . -a cursor
npx skills list
```

## When it applies

Use when the user or agent mentions:

- Migrating Next.js to Astro
- Removing `next.config.js`, `app/`, or `pages/`
- Replacing `next/*` imports with Astro patterns
- Moving routes to `src/pages/*.astro`

The skill auto-activates from its `description` frontmatter in [SKILL.md](./SKILL.md).

## Prerequisites

- **Node.js 22+** (required by Astro 7)
- **Astro 7** (`astro@^7`)
- An existing Next.js 13, 14, 15, or 16 project

## Package layout

This repository **is** the skill package. [SKILL.md](./SKILL.md) lives at the repo root (valid for skills CLI discovery).

| File | Purpose |
|------|---------|
| [SKILL.md](./SKILL.md) | Core instructions — required `name` + `description` frontmatter |
| [planning-rubric.md](./planning-rubric.md) | Phase 0 plan scrutiny |
| [mapping.md](./mapping.md) | Route, API, env, and config lookup |
| [conversion-rubric.md](./conversion-rubric.md) | Component → `.astro` / island decision tree |
| [nextjs-versions.md](./nextjs-versions.md) | Next 13–16 version-specific flags |

Supporting `.md` files ship with the skill when installed via `npx skills add`.

## Official Astro docs

- [Migrating from Next.js](https://docs.astro.build/en/guides/migrate-to-astro/from-nextjs/)
- [Upgrade to Astro v7](https://docs.astro.build/en/guides/upgrade-to/v7/)

## Out of scope

- **CSS / Tailwind / styling framework migration** — carry over the existing stack as-is
- **Hosting and deploy** — adapters, CDN rewrites, artifact packaging (stop when `astro build` passes locally)

## License

MIT — see [LICENSE](./LICENSE).
