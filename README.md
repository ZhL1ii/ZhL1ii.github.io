# ZhL1ii

A personal blog built with Astro, Tailwind CSS, Markdown/MDX content
collections, RSS, sitemap, Pagefind search, and generated Open Graph images.

## Project Structure

```bash
src/content/pages/about.md      # static about page content
src/content/posts/              # blog posts
site.config.ts                  # site metadata, features, social links
astro.config.ts                 # Astro integrations and build config
```

## Configuration

Edit `site.config.ts` to update the site title, deployed URL, author,
timezone, social links, sharing links, and feature flags.

## Writing

Create new Markdown or MDX files in `src/content/posts/`.

Required frontmatter:

```yaml
---
title: Post title
pubDatetime: 2026-06-11T00:00:00Z
description: Short post summary.
---
```

Optional fields include `author`, `modDatetime`, `featured`, `draft`, `tags`,
`ogImage`, `canonicalURL`, `hideEditPost`, and `timezone`.

## Commands

| Command        | Action                                          |
| :------------- | :---------------------------------------------- |
| `pnpm install` | Install dependencies                            |
| `pnpm dev`     | Start the local dev server                      |
| `pnpm build`   | Type-check, build, index search, and emit files |
| `pnpm preview` | Preview the production build                    |
| `pnpm sync`    | Generate Astro types                            |
| `pnpm lint`    | Run ESLint                                      |
