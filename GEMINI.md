# Gemini Project Context: ssssshql.github.io

## Project Overview
This is a personal blog built with the **Hexo** static site generator using the **Cactus** theme. It is primarily used for technical notes, software installation guides, and personal records.

- **Primary Technologies:** Hexo (v8.1.1), Node.js, Markdown, EJS (templates), Stylus (CSS).
- **Theme:** [Cactus](https://github.com/probberechts/hexo-theme-cactus), a responsive, clean and simple Hexo theme.
- **Content Language:** Chinese (`zh-CN`).

## Directory Structure
- `source/_posts/`: Contains all blog posts in Markdown format.
- `themes/cactus/`: The source code and configuration for the blog theme.
- `_config.yml`: Global Hexo configuration.
- `_config.cactus.yml`: Theme-specific configuration (Cactus theme).
- `.codebuddy/rules/`: Custom AI instructions for writing posts.
- `scaffolds/`: Templates for new posts, pages, and drafts.

## Building and Running
The project uses **pnpm** as the package manager (though `npm` scripts are available).
The following commands are defined in `package.json`:

- **Development Server:** `npm run server` (starts `hexo server` at `http://localhost:4000`)
- **Generate Static Files:** `npm run build` (runs `hexo generate`)
- **Clean Public Files:** `npm run clean` (runs `hexo clean`)
- **Deploy:** `npm run deploy` (runs `hexo deploy`)

## Development & Blogging Conventions

### Writing Style
As per the project's custom rules (`.codebuddy/rules/写博客.mdc`):
- **Style:** "随手记" (informal, note-taking style).
- **Visuals:** Use image placeholders during the drafting process.

### Post Creation
- To create a new post: `npx hexo new post "Title"`
- Filename convention: `:title.md` (e.g., `my-new-post.md`).
- Posts should include front-matter (title, date, tags, categories).

### Technical Integrity
- When modifying the theme (`themes/cactus`), ensure compatibility with EJS and Stylus.
- Always run `hexo clean` before `hexo generate` if there are significant layout or configuration changes to ensure a fresh build.
