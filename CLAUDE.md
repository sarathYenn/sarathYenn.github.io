# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A static blog for [blog.arcturanlabs.com](https://blog.arcturanlabs.com), built with Astro 4. Posts are Markdown files; Astro renders them at build time into a static site deployed to GitHub Pages via GitHub Actions on every push to `main`.

## Commands

| Action | Command |
|---|---|
| Dev server | `npm run dev` → http://localhost:4321 |
| Production build | `npm run build` → `dist/` |
| Preview built site | `npm run preview` |

No test suite, no linter configured.

## Architecture

All styling lives in `<style>` blocks scoped to each `.astro` file — there is no external CSS framework or global stylesheet. CSS variables for the color system (`--bg`, `--surface`, `--border`, `--text`, `--muted`, `--accent`, `--accent-light`) are declared in `BaseLayout.astro` and inherited by every page.

Layout hierarchy:
- `BaseLayout.astro` — full HTML shell (head, header nav, footer, CSS variables). Accepts `title` and `description` props.
- `PostLayout.astro` — wraps `BaseLayout`; reads `frontmatter` from Astro.props and renders the post header (date, tags, title, description) above a `<slot />` for the Markdown body.

Index page (`src/pages/index.astro`) collects all posts via `Astro.glob('./posts/*.md')` and sorts by `frontmatter.date` descending.

## Adding a post

Create `src/pages/posts/<slug>.md` with this frontmatter:

```markdown
---
layout: ../../layouts/PostLayout.astro
title: "Post Title"
description: "One-sentence summary shown on the index."
date: YYYY-MM-DD
tags: ["tag-one", "tag-two"]
---
```

The index page picks it up automatically — no registration needed. The `layout` path must be exactly `../../layouts/PostLayout.astro` (two levels up from `posts/`).

## Writing style for posts

- **No real project examples.** Do not reference specific companies, codebases, products, or incidents by name. All examples must be generic (e.g. "a payment service", "a data pipeline", "a team I've seen") as if the author synthesized observations from broad research rather than personal production experience.
- **No literal code samples.** Explain concepts in prose. If a concrete illustration is needed, use pseudocode or a brief abstract snippet — never a real function, class, or config block that looks copy-pasteable.
- **Narration voice.** Write as someone who has studied a problem space deeply and is sharing distilled findings — not as someone debugging their own system. Phrases like "teams that do X tend to…" or "a common pattern is…" fit the voice; "at my last job we…" does not.
- **Keep examples clearly illustrative.** When an example is necessary, signal that it is hypothetical ("imagine a service that…", "consider a team where…").

## Deployment

Pushing to `main` triggers `.github/workflows/deploy.yml`, which builds with Node 20 and deploys `dist/` to GitHub Pages. The custom domain `blog.arcturanlabs.com` is set via `public/CNAME` — do not delete or modify that file.
