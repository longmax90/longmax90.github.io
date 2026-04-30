# Software Security Blog in C/C++/Rust

GitHub Pages blog for publishing software security articles on C, C++, and Rust.

**Live Site:** https://longmax90.github.io

---

## Quick Start

```bash
cd /home/longmax/blogs
bundle install --local
bundle exec jekyll serve --livereload
```

Visit **http://127.0.0.1:4000** in your browser.

---

## Setup & Installation

### Prerequisites

- Ruby 3.2+: `sudo apt-get install ruby3.2 ruby3.2-dev`
- Build tools: `sudo apt-get install build-essential git`
- Bundler: `gem install bundler`

### Install Dependencies

```bash
bundle init
bundle add github-pages --group "jekyll_plugins"
bundle config set --local path vendor/bundle
bundle install
```

---

## Local Development

### Start Server

```bash
bundle exec jekyll serve --livereload
```

Access at **http://127.0.0.1:4000**. Changes auto-reload; press **Ctrl+C** to stop.

---

## Writing Posts

Post files go in `_posts/` with format: `YYYY-MM-DD-post-slug.md`

### Front Matter Template

```yaml
---
layout: default
title: Post Title
date: 2026-04-30 00:00:00 +0000
categories: [category-slug]
tags: [tag1, tag2]
---
```

Write content in Markdown after front matter. URLs are built as:
`/:categories/:year/:month/:day/:title/`

---

## Publishing

### Before Push

1. Start local server and verify post renders correctly
2. Check links and formatting

### Commit & Deploy

```bash
git add .
git commit -m "Add post \"[Title]\""
git push origin main
```

GitHub Pages auto-deploys in 1–5 minutes. Site updates at https://longmax90.github.io

---

## Project Structure

```
├── _posts/                # Blog articles
├── _layouts/default.html  # Page template
├── assets/css/site.css    # Styling
├── categories.md          # Category index
├── tags.md               # Tag index
├── index.md              # Homepage
└── _config.yml           # Jekyll config
```

---

## Editorial Guidelines

See `.github/copilot-instructions.md` for project conventions and writing standards. All posts must be rigorous, evidence-driven technical content with reproducible claims and clear threat models.

---

**Last Updated:** April 30, 2026
