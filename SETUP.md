# Arcturan Labs Blog — Setup Guide

## 1. Create a GitHub repo

Go to github.com and create a new repo named exactly:
```
sarathYenn.github.io
```
This is the magic name that activates GitHub Pages for free at `https://sarathYenn.github.io`.

## 2. Clone and set up locally

```bash
# Move the project folder to wherever you keep code
mv arcturan-labs-blog ~/code/sarathYenn.github.io
cd ~/code/sarathYenn.github.io

# Install dependencies
npm install

# Preview locally
npm run dev
# → opens at http://localhost:4321
```

## 3. Push to GitHub

```bash
git init
git add .
git commit -m "initial blog setup"
git branch -M main
git remote add origin git@github.com:sarathYenn/sarathYenn.github.io.git
git push -u origin main
```

## 4. Enable GitHub Pages

1. Go to your repo → **Settings** → **Pages**
2. Under **Source**, select **GitHub Actions**
3. That's it — the deploy workflow runs automatically on every push

Your blog will be live at `https://sarathYenn.github.io` within ~2 minutes.

## 5. Writing a new post

Create a file in `src/pages/posts/`:

```bash
touch src/pages/posts/my-new-post.md
```

Start the file with this frontmatter:

```markdown
---
layout: ../../layouts/PostLayout.astro
title: "Your Post Title"
description: "One sentence summary for the index page."
date: 2026-06-23
tags: ["distributed-systems", "reliability"]
---

Your content here in Markdown...
```

Then:
```bash
git add .
git commit -m "post: your post title"
git push
```

GitHub Actions deploys it automatically. Done.

## Project structure

```
.
├── src/
│   ├── layouts/
│   │   ├── BaseLayout.astro   # Site shell (header, footer, styles)
│   │   └── PostLayout.astro   # Individual post layout
│   └── pages/
│       ├── index.astro        # Post listing
│       ├── about.astro        # About page
│       └── posts/
│           └── *.md           # Your blog posts go here
├── .github/workflows/
│   └── deploy.yml             # Auto-deploy on push to main
├── astro.config.mjs
└── package.json
```
