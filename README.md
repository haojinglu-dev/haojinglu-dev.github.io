# haojing.dev

A minimal engineering blog built with Astro. Articles are Markdown files and
the site is ready for GitHub and Cloudflare Pages.

## Run locally

```bash
npm install
npm run dev
```

## Customize

- Edit the homepage bio in `src/pages/index.astro`.
- Add articles to `src/content/blog/`.
- Edit profile links in `src/consts.ts`.
- Adjust the design in `src/styles/global.css`.

Create a production build with `npm run build`.

## Publish an article

Create a Markdown file in `src/content/blog`:

```md
---
title: "Article title"
description: "A one-sentence summary."
pubDate: 2026-08-20
tags:
  - distributed-systems
---

Your article goes here.
```
