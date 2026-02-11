# wpmigrate-skills

Agent skills for migrating WordPress sites. Provides structured guides and best practices that AI coding agents can reference when helping users extract content from WordPress and convert it to static site generators.

Built from the real-world migration of [andrewmiracle.com](https://andrewmiracle.com) — 429 WordPress pages converted to a Jekyll site with dark mode, digital garden features, and RSS feeds.

## Installation

```bash
npx skills add https://github.com/koolamusic/wpmigrate-skills
```

## Skills

| Skill | Description |
|-------|-------------|
| **wordpress-migration-best-practices** | General best practices for migrating content out of WordPress to any platform |
| **wp-to-jekyll** | Step-by-step guide for converting WordPress content into a Jekyll static site |

### wordpress-migration-best-practices

Use when asked to "migrate from WordPress", "export WordPress content", "move off WordPress", or "extract WordPress data".

Covers:
- 4 extraction strategies (XML export, site mirroring, combined approach, direct database)
- Plugin-specific content handling (WooCommerce, Contact Forms, Page Builders, SEO plugins, ACF)
- Media migration and URL structure preservation
- Pre/during/post migration checklists

```bash
npx skills add https://github.com/koolamusic/wpmigrate-skills --skill wordpress-migration-best-practices
```

### wp-to-jekyll

Use when asked to "convert WordPress to Jekyll", "migrate WP to Jekyll", "WordPress to static site", or "export WordPress to markdown".

Covers:
- Quick start with the [jekyllwind](https://github.com/koolamusic/jekyllwind) starter template
- 6-phase migration process (extraction, cleanup, architecture, design, features, deployment)
- 19-step content cleanup pipeline for WordPress artifacts
- Jekyll collections, layouts, and Tailwind CSS integration
- Deployment on Netlify with URL redirects and RSS feeds

```bash
npx skills add https://github.com/koolamusic/wpmigrate-skills --skill wp-to-jekyll
```

## When to Use

Reference these skills when:
- Planning a migration away from WordPress
- Converting WordPress content to a static site generator
- Dealing with complex WordPress sites (WooCommerce, page builders, custom plugins)
- Cleaning up WordPress markup artifacts (Gutenberg blocks, Visual Composer, inline styles)
- Setting up Jekyll with Tailwind CSS for a migrated site

## Migration Overview

The root `SKILL.md` contains a comprehensive playbook covering the full migration pipeline:

1. **Source Acquisition** — Site mirroring with HTTrack/wget to capture rendered HTML and media
2. **Content Extraction** — Python scripts to parse HTML, classify content, and generate frontmatter
3. **Content Cleanup** — 19-step pipeline to strip Gutenberg comments, Visual Composer nesting, and WordPress artifacts
4. **Jekyll Architecture** — Collections, layouts, plugins, and permalink structure
5. **Design System** — Tailwind CSS with custom tokens, dark mode, and typography
6. **Feature Layer** — Digital garden stages, password protection, tag filtering
7. **Build & Deployment** — Netlify configuration, URL redirects, RSS feeds

## Project Structure

```
├── SKILL.md                                   # Full migration playbook
└── skills/
    ├── wordpress-migration-best-practices/
    │   └── SKILL.md                           # General WP migration guide
    └── wp-to-jekyll/
        └── SKILL.md                           # WP-to-Jekyll conversion guide
```

## License

MIT
