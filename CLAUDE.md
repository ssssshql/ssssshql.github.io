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

## Writing posts (house style)

This repo’s posts are primarily Chinese and often written in a casual “随手记” style (see `.codebuddy/rules/写博客.mdc`). When drafting posts, optimize for skim-readability and copy/paste-able commands.

Patterns observed in existing posts (`source/_posts/*.md`):

- **Open with context**: 1–3 short paragraphs on what you did/why (first person is normal), then get into steps.
- **Use clear sectioning**:
  - Guides: `简介/前置条件/步骤/验证/总结`
  - Troubleshooting: `现象(错误信息)/原因/解决方案(方案一/二)/补充说明/参考链接`
- **Make it easy to scan**:
  - Prefer short paragraphs, lists, and numbered steps.
  - Use **bold** for key constraints/"坑"/cost numbers.
  - Quote raw error messages with `>`.
- **Commands should be runnable**: put shell commands in fenced code blocks with the right language (e.g. `bash`, `powershell`). Keep lines minimal and avoid wrapping.
- **Images**:
  - In drafts, include placeholders like `![这里放截图说明]()` where a screenshot/diagram helps.
  - Final posts often embed images via `https://picgo.19991029.xyz/...` and sometimes use HTML `<img ... style="zoom:...">` for side-by-side or size control.
- **Front-matter**: always include `title` + `date`. Tags/categories are commonly used and can be inline arrays (`tags: [a, b]`) or YAML lists.

### User writing preferences (explicit)

- Use the **actual current local time** for post `date`; avoid fixed placeholder times.
- In troubleshooting posts, explicitly突出“真正踩坑点/根因”（例如权限所有者是 `TrustedInstaller`），避免只给表面操作步骤。
- Keep room for screenshots in key sections; final images are often added manually with `https://picgo.19991029.xyz/...` links.

