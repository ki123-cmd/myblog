# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **Hugo static site** using the **hugo-theme-next** (NexT for Hugo) theme. The theme is loaded as a git submodule under `themes/hugo-theme-next`. There is also a `themes/blowfish` submodule (currently inactive).

The active theme is configured in `hugo.yaml` (`theme: hugo-theme-next`). Theme submodule updates and customization require understanding Hugo's lookup order: custom layouts in `layouts/` override the theme's `layouts/`.

## Commands

### Development
- `hugo server` — Start local dev server (http://localhost:1313)
- `hugo server -D` — Start server including draft posts
- `hugo` — Full static build to `public/`
- `hugo --gc` — Build with garbage collection of unused caches

### Theme Management
- `git submodule update --remote themes/hugo-theme-next` — Pull latest theme changes
- `git submodule update --remote themes/blowfish` — Update blowfish submodule

### Content
- Create a new post: place a `.md` file under `content/post/`
- Use the archetype template in `archetypes/default.md` as a starting point
- Draft posts: add `draft: true` in front matter

## Architecture

### Layout Override System
Hugo resolves templates in this order (highest wins):
1. `layouts/` — your custom overrides
2. `themes/hugo-theme-next/layouts/` — the NexT theme
3. Built-in Hugo defaults

This site already has custom overrides:
- `layouts/index.html` — homepage override
- `layouts/partials/custom_footer.html` — custom footer content
- `layouts/partials/custom_sidebar.html` — custom sidebar content

To override any theme template, copy the relevant file from the theme to your `layouts/` directory and modify it there.

### Configuration
- `hugo.yaml` — main Hugo config (bilingual Chinese/English comments throughout)
- Key config sections in `hugo.yaml`:
  - `params.mainSections: ["post"]` — content sections shown on home/archive
  - `params.scheme` — page layout (Muse, Mist, Pisces, Gemini; currently Gemini)
  - `params.darkmode` — dark mode toggle
  - `params.localSearch` — client-side search (Fuse.js), outputs to `/searchindexes.xml`
  - `params.waline` — comment system (requires `serverURL`)
  - `params.postMeta` — per-post metadata display (views, word count, read time)
  - `params.codeblock.style` — code block style (default, flat, mac)
  - `markup.goldmark.renderer.unsafe: true` — allows raw HTML in Markdown
  - `markup.highlight` — code highlighting settings

### Theme Structure (themes/hugo-theme-next)
- `layouts/_markup/` — render hooks for Markdown hooks (heading anchors, link icons, etc.)
- `layouts/_partials/` — page section partials (header, footer, sidebar, post items)
- `layouts/_shortcodes/` — custom shortcodes
- `layouts/archives`, `layouts/home.html`, `layouts/list.html` — page type templates
- `layouts/baseof.html` — base HTML skeleton

### Content Structure
- `content/post/` — blog posts (main section)
- `content/authors/` — multi-author support (NexT feature)
- Post front matter supports: `title`, `date`, `draft`, `tags`, `categories`, `description`, `toc`, `reward`

## Notes
- The theme `hugo-theme-next` is a fork of the original NexT theme adapted for Hugo. It is NOT the same as hexo-theme-next.
- Custom CSS can be added via `params.customFilePath.style` in `hugo.yaml`.
- The theme uses Font Awesome 6 for icons, KaTeX/Mermaid/Charts for rich content.
- Chinese/Japanese/Korean language detection is enabled (`hasCJKLanguage: true`).
