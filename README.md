# My Technical Blog

A clean, simple, and fast personal blog where I share ideas, notes, and longer technical write-ups. This repository contains the source for my blog and the configuration needed to run it locally and deploy it to the web.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Made with Markdown](https://img.shields.io/badge/Markdown-✓-blue)](#writing-posts)


## Contents
- Overview
- Features
- Quick start
- Writing posts
- Local development
- Deployment options
- Project structure
- Customization
- SEO and feeds
- Contributing
- License
- Contact


## Overview
This is my technical blog where I publish tutorials, notes, and opinions about software engineering and related topics. The repository is intentionally simple so I can focus on writing rather than tooling. It works well with GitHub Pages and most static site generators (Jekyll, Hugo, Astro, 11ty, etc.).

If you are visiting the code first: the live site URL will typically be one of the following (replace with your actual link):
- https://<your-username>.github.io/
- https://<your-username>.github.io/<repo-name>/
- A custom domain configured via DNS (e.g., https://blog.example.com)


## Features
- Write posts in plain Markdown (.md) with optional front matter
- Syntax highlighting for code blocks (depends on chosen SSG/theme)
- Tags/categories and archives (via SSG configuration)
- Works with GitHub Pages, Netlify, or Vercel
- Versioned content: every change is tracked in Git
- MIT licensed


## Quick start
You can use this repository with your preferred static site generator (SSG) or GitHub Pages. Pick one of the options below.

Option A — GitHub Pages (Jekyll under the hood):
1) Enable GitHub Pages in your repository settings (Pages → Source → GitHub Actions or Deploy from branch).
2) Add a basic Jekyll config (_config.yml) and a homepage if needed. GitHub Pages can build sites without extra tooling.
3) Commit and push; GitHub will publish your site at the configured URL.

Option B — Use your favorite SSG locally and deploy:
1) Choose an SSG: Jekyll (Ruby), Hugo (Go), 11ty (Node.js), Astro (Node.js), etc.
2) Scaffold your site with the SSG’s CLI, or adapt the structure here.
3) Run locally, iterate, then deploy to GitHub Pages, Netlify, or Vercel.


## Writing posts
Create posts as Markdown files. If your SSG supports front matter, use YAML at the top for metadata. Example:

---
title: "Understanding Async/Await in JavaScript"
date: 2025-09-26
description: "A practical walk-through with gotchas."
tags: [javascript, async]
cover_image: /images/async-cover.png
canonical_url: https://example.com/async-await-guide
---

Write your content in Markdown below the front matter. Use fenced code blocks for snippets:

```js
async function fetchJson(url) {
  const res = await fetch(url);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}
```

Tips:
- Prefer short paragraphs and descriptive headings
- Add diagrams or screenshots when helpful
- Use canonical_url if you cross-post


## Local development
Depending on the SSG you choose, the commands differ. Common options:

Jekyll (Ruby):
- Prereqs: Ruby + Bundler
- Commands: bundle install; bundle exec jekyll serve

Hugo (Go):
- Prereqs: Hugo binary
- Commands: hugo server

11ty (Node.js):
- Prereqs: Node.js + npm
- Commands: npm install; npx @11ty/eleventy --serve

Astro (Node.js):
- Prereqs: Node.js + npm
- Commands: npm install; npm run dev


## Deployment options
- GitHub Pages: Free and easy; good default. Use GitHub Actions or Pages build.
- Netlify: Connect repo; set build command (e.g., hugo, eleventy, astro build) and publish directory.
- Vercel: Great for Node-based SSGs (Astro/Next). Import repo and deploy.

If using a custom domain, set DNS (CNAME/ALIAS) and ensure the platform has the domain configured with HTTPS enabled.


## Project structure
This repository is intentionally minimal. A typical blog structure might look like:

.
├── content/ or _posts/            # your Markdown posts
├── assets/ or static/             # images, fonts, etc.
├── layouts/ or themes/            # templates and partials (SSG-dependent)
├── config files (e.g., _config.yml, astro.config.mjs, .eleventy.js)
├── LICENSE                        # license for the repo
└── README.md                      # this file

Your actual structure will depend on the SSG/theme you choose.


## Customization
- Theme: pick a starter theme for your SSG or create your own styles
- Navigation: define header/footer links in your config or templates
- Code highlighting: enable your preferred highlighter (Rouge, Prism, Shiki)
- Images: optimize and add alt text for accessibility
- Analytics: add your preferred analytics snippet (e.g., Plausible, GA)


## SEO and feeds
- Add a sitemap.xml and robots.txt (plugins or SSG options)
- Generate an RSS/Atom feed so readers can subscribe
- Configure meta tags and Open Graph/Twitter cards for social sharing


## Contributing
While this is primarily my personal blog, I welcome suggestions and issue reports. Feel free to open an issue for typos, broken links, or ideas.


## License
This project is licensed under the MIT License. See the LICENSE file for details.


## Contact
- Author: Valentyn Yakymenko
- Medium: https://medium.com/@vyakymenko
- Twitter/X: https://x.com/valeyakymenko
- LinkedIn: https://www.linkedin.com/in/vale-yakymenko/
- Email: vale.yakymenko@gmail.com

If you build your own blog from this, I’d love to see it—share a link!
