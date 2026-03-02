# Stefano Mondino - Hugo Site

Personal website built with Hugo and the [Mana](https://github.com/Livour/hugo-mana-theme) theme.

## Stack

- **Hugo** v0.157+ (Extended) - managed via `mise`
- **Node** 22 - managed via `mise`
- **Theme**: Mana (git submodule in `themes/mana`)

## Commands

- `mise exec -- hugo server -D` — start the dev server (with drafts)
- `mise exec -- hugo` — build the site into `public/`
- `mise install` — install dependencies (hugo + node) via mise

## Project Structure

```
hugo.toml              # Main site configuration
mise.toml              # Tool version management (hugo, node)
content/
  posts/               # Blog articles (markdown)
  about/index.md       # About page
static/                # Static files (favicon, unprocessed images)
assets/images/         # Hugo-processed images (resize, optimization)
themes/mana/           # Mana theme (git submodule - DO NOT modify directly)
```

## Content Conventions

- Posts go in `content/posts/` as `.md` files
- Required front matter: `title`, `date`, `draft`, `tags`
- The About page uses the `about` layout and lives at `content/about/index.md`
- Default language: English (`en`)

## Mana Theme - Notes

- The theme is a git submodule: never modify files inside `themes/mana/`
- To customize layouts, create overrides in `layouts/` at the project root
- To customize CSS, create files in `assets/css/` at the root
- Supports GitHub/Obsidian-style admonitions (NOTE, TIP, WARNING, etc.)
- Supports dark/light mode toggle
- Full-text search requires JSON output (already configured in `hugo.toml`)

## Configuration

- `hugo.toml` contains all configuration: menu, params, social links
- Single language (English), single content folder
- Syntax highlighting with Catppuccin themes

## Rules for Claude

- Always use `mise exec -- hugo` to run hugo commands
- Never modify files inside `themes/mana/`
- To override templates, copy the file from `themes/mana/layouts/` to `layouts/` keeping the same path
- Content is in English unless otherwise specified
- After changes to configuration or layouts, verify with `mise exec -- hugo` that the build works
