# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project summary

This repo is a Hexo static blog (Hexo 8.x) deployed to GitHub Pages via GitHub Actions. Content lives in `source/` (mostly Chinese), and the site uses the `themes/cactus` theme with overrides in `_config.cactus.yml`.

## Common commands

### Install

- Install dependencies (repo root):
  - `pnpm install`

> CI uses Node 20 + pnpm.

### Local development

- Start dev server (default: http://localhost:4000):
  - `pnpm run server`

- Generate static site into `public/`:
  - `pnpm run build`

- Clean generated output:
  - `pnpm run clean`

### Create content

- New post:
  - `pnpm exec hexo new post "Title"`

- New page:
  - `pnpm exec hexo new page about`

Scaffolds used for new content are in `scaffolds/`.

### Theme checks (Cactus)

The theme has its own Node toolchain under `themes/cactus/` (gulp-based lint/validate).

- Install theme dev deps:
  - `pnpm -C themes/cactus install`

- Lint theme assets:
  - `pnpm -C themes/cactus run lint`

- Validate theme YAML ("tests"):
  - `pnpm -C themes/cactus run test`

- Run a single gulp task (examples):
  - `pnpm -C themes/cactus exec gulp validate:config`
  - `pnpm -C themes/cactus exec gulp validate:languages`
  - `pnpm -C themes/cactus exec gulp lint:js`
  - `pnpm -C themes/cactus exec gulp lint:stylus`

## Deployment (GitHub Pages)

- Workflow: `.github/workflows/deploy.yml`
- Trigger: push to `main`
- Build: `pnpm install` then `pnpm build` (Hexo generates `./public`)
- Deploy: uploads `./public` as Pages artifact and deploys it

Note: `_config.yml` has `deploy.type: ''` — publishing is handled by GitHub Actions rather than `hexo deploy`.

## High-level structure

- `_config.yml`: main Hexo config (site-wide settings).
- `_config.cactus.yml`: **theme override config** for the `cactus` theme. Prefer editing this instead of `themes/cactus/_config.yml`.
- `source/_posts/`: blog posts (Markdown).
- `source/<page>/index.md`: pages like `about/`, `tags/`, `categories/`, `archives/`, `search/`.
- `themes/cactus/`: Cactus theme source (EJS layouts under `layout/`, Stylus under `source/css/`, i18n under `languages/`).
- `public/`: generated output (ignored by git).

## Repo-specific authoring rule

There is a CodeBuddy rule at `.codebuddy/rules/写博客.mdc`: when asked to write blog content, use a casual “随手记” note style and you may include image placeholders for later insertion.
