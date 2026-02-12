# wpmigrate-skills

Agent skills for migrating WordPress sites. Provides structured guides and best practices that AI coding agents can reference when helping users extract content from WordPress and convert it to static site generators.

Built from the real-world migration of [andrewmiracle.com](https://andrewmiracle.com) — 429 WordPress pages converted to a Jekyll site with dark mode, digital garden features, and RSS feeds.

## Setup

### Install all skills

```bash
npx skills add https://github.com/koolamusic/wpmigrate-skills
```

### Install a single skill

```bash
# General WordPress migration best practices
npx skills add https://github.com/koolamusic/wpmigrate-skills --skill wordpress-migration-best-practices

# WordPress-to-Jekyll conversion guide
npx skills add https://github.com/koolamusic/wpmigrate-skills --skill wp-to-jekyll
```

### Verify installation

```bash
npx skills check
```

After installation, the skills are automatically available to your AI coding agent (Claude Code, Cursor, Windsurf, etc.). No additional configuration is needed — the agent will reference the skills when it detects a relevant request.

## Usage

Once installed, ask your agent to help with WordPress migration tasks. The skills activate automatically based on your prompt.

### Example prompts

**General migration planning:**
- "I need to migrate my WordPress blog to a static site. What's the best approach?"
- "How do I export content from a WordPress site that uses WooCommerce and Elementor?"
- "What's the best way to handle media files during a WordPress migration?"
- "Help me plan a migration strategy for my WordPress site with 500+ posts"

**WordPress to Jekyll:**
- "Convert my WordPress site to Jekyll"
- "Set up a Jekyll project from my WordPress XML export"
- "Clean up the WordPress HTML artifacts in my migrated content"
- "Help me configure Jekyll collections to match my WordPress custom post types"
- "Deploy my migrated Jekyll site to Netlify"

### What the agent gets

When a skill is triggered, the agent receives detailed, structured guidance including:

- Step-by-step migration procedures
- Code snippets for content extraction and cleanup (Python, Bash, Ruby)
- Configuration templates (`_config.yml`, `netlify.toml`, `Gemfile`)
- Decision trees for choosing extraction strategies
- Pitfalls and lessons learned from real migrations

### Prerequisites for migration projects

The skills reference the following tools. Install what you need based on your migration approach:

| Tool | Version | Purpose |
|------|---------|---------|
| Ruby | 3.2+ | Jekyll runtime |
| Bundler | latest | Ruby dependency management |
| Node.js | 18+ | Tailwind CSS compilation |
| pnpm | latest | Node package manager |
| Python 3 | 3.8+ | Content extraction and cleanup scripts |
| BeautifulSoup4 | latest | HTML parsing (`pip install beautifulsoup4`) |
| HTTrack or wget | latest | Site mirroring (optional) |

## Skills

| Skill | Description |
|-------|-------------|
| **wordpress-migration-best-practices** | General best practices for migrating content out of WordPress to any platform |
| **wp-to-jekyll** | Step-by-step guide for converting WordPress content into a Jekyll static site |

### wordpress-migration-best-practices

Triggered when users ask to "migrate from WordPress", "export WordPress content", "move off WordPress", or "extract WordPress data".

Covers:
- 4 extraction strategies and when to use each (XML export, site mirroring, combined approach, direct database)
- Plugin-specific content handling (WooCommerce, Contact Forms, Page Builders, SEO plugins, ACF)
- Media migration and URL structure preservation
- Pre/during/post migration checklists

### wp-to-jekyll

Triggered when users ask to "convert WordPress to Jekyll", "migrate WP to Jekyll", "WordPress to static site", or "export WordPress to markdown".

Covers:
- Quick start with the [jekyllwind](https://github.com/koolamusic/jekyllwind) starter template
- 6-phase migration process (extraction, cleanup, architecture, design, features, deployment)
- 19-step content cleanup pipeline for stripping WordPress artifacts
- Jekyll collections, layouts, and Tailwind CSS integration
- Deployment on Netlify and GitHub Pages with URL redirects and RSS feeds
- 10 documented pitfalls from a real 429-page migration

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
├── SKILL.md                                   # Full migration playbook (679 lines)
└── skills/
    ├── wordpress-migration-best-practices/
    │   └── SKILL.md                           # General WP migration guide
    └── wp-to-jekyll/
        └── SKILL.md                           # WP-to-Jekyll conversion guide
```

## License

MIT
