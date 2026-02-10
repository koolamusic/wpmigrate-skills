# WordPress-to-Jekyll Migration Playbook

A reusable guide for migrating a WordPress site (via static HTML clone) to Jekyll 4.4 + Tailwind CSS 3.4, deployed on Netlify. Based on the migration of [andrewmiracle.com](https://andrewmiracle.com) — 429 static HTML pages spanning 2012–2025, converted to a fully themed Jekyll site with dark mode, digital garden features, password-protected projects, and RSS feeds.

**Source:** 429 WordPress pages scraped to static HTML
**Target:** Jekyll 4.4.1 + Tailwind 3.4.17 + PostCSS
**Deployment:** Netlify (Ruby 3.2, Node 18)
**Output:** 178 blog posts, 50 lab projects, 10 talks, 3 notes, 8 drafts, 6 standalone pages

---

## Prerequisites

### System Dependencies

| Tool | Version | Purpose |
|------|---------|---------|
| Ruby | 3.2+ | Jekyll runtime |
| Bundler | latest | Ruby dependency management |
| Node.js | 18+ | Tailwind CSS compilation |
| pnpm | latest | Fast Node package manager |
| Python 3 | 3.8+ | Content extraction and cleanup scripts |
| BeautifulSoup4 | latest | HTML parsing in Python scripts |

### Ruby Gems (Gemfile)

```ruby
gem 'jekyll'            # Static site generator
gem 'webrick'           # Dev server (Ruby 3.x dropped it from stdlib)
gem 'jekyll-postcss-v2' # PostCSS/Tailwind integration
gem 'jekyll-feed'       # RSS/Atom feed generation
gem 'jekyll-sitemap'    # XML sitemap generation
gem 'jekyll-seo-tag'    # Meta tags and structured data
gem 'logger'            # Ruby 3.x stdlib extraction
gem 'csv'               # Ruby 3.x stdlib extraction
gem 'base64'            # Ruby 3.x stdlib extraction
```

### npm Packages (package.json)

```json
{
  "devDependencies": {
    "tailwindcss": "^3.4.17",
    "@tailwindcss/typography": "^0.5.19",
    "autoprefixer": "^10.4.7",
    "postcss": "^8.5.6",
    "postcss-cli": "^11.0.1"
  }
}
```

### Bootstrap Commands

```bash
bundle install && pnpm install    # Install all dependencies
bundle exec jekyll serve          # Dev server at localhost:4000
bundle exec jekyll build          # Production build to _site/
bundle exec jekyll build --drafts # Include _drafts/ content
```

---

## Phase 1 — Source Acquisition

Scrape the live WordPress site to a local static HTML clone using HTTrack or wget. The goal is a complete mirror preserving the original URL structure.

### Approach

```bash
# HTTrack example — mirror entire site
httrack "https://andrewmiracle.com" -O ./andrewmiracle.com \
  --mirror --robots=0 --depth=10

# Alternative with wget
wget --mirror --convert-links --adjust-extension \
  --page-requisites --no-parent https://andrewmiracle.com
```

### What You Get

```
andrewmiracle.com/
├── index.html                      # Homepage
├── 2025/02/26/post-slug/index.html # Blog posts (YYYY/MM/DD/slug/)
├── lab/project-name/index.html     # Portfolio projects
├── talks/talk-name/index.html      # Speaking engagements
├── whoami/index.html               # Static pages
├── wp-content/uploads/             # All media files
└── ...                             # 429 HTML files total
```

### Key Details

- WordPress uses `YYYY/MM/DD/slug/` permalink structure — preserve this exactly
- Images land in `wp-content/uploads/YYYY/MM/` — these get reorganized later
- Some projects may be password-protected (WordPress server-side auth) — the scraper captures only the public-facing page, not protected content
- Check for duplicate pages (e.g., `/embed/` variants of posts) that can be skipped

---

## Phase 2 — Content Extraction

A Python script parses each scraped HTML file, classifies it by URL pattern, extracts frontmatter from meta tags, pulls body content from WordPress DOM structure, and writes Jekyll-compatible files.

### Script: `scripts/extract.py`

**Classification logic** — route files to the correct Jekyll collection based on URL path:

| URL Pattern | Output | Collection |
|-------------|--------|------------|
| `/YYYY/MM/DD/slug/` | `_posts/YYYY-MM-DD-slug.html` | Blog posts |
| `/lab/slug/` | `_lab/slug.html` | Portfolio projects |
| `/talks/slug/` | `_talks/slug.html` | Speaking engagements |
| `/whoami/`, `/now/`, etc. | `pages/slug.html` | Standalone pages |
| Everything else | Skip or manual review | — |

**Frontmatter extraction** — derive YAML frontmatter from HTML `<meta>` tags:

```python
# OpenGraph tags → frontmatter fields
<meta property="og:title">       → title
<meta property="og:description"> → description
<meta property="og:image">       → image
<meta property="og:url">         → permalink

# Article tags → frontmatter fields
<meta property="article:published_time"> → date
<meta property="article:modified_time">  → last_modified_at
<meta property="article:tag">            → tags (multiple)
<meta property="article:section">        → categories
```

**Content extraction** — isolate the article body from WordPress page chrome:

```python
# Target: div.post-content between header.entry-header and footer.article-tags
# This strips navigation, sidebars, related posts, comments, and footer
content = soup.find("div", class_="post-content")
```

**Image handling:**

1. Prefer `data-src` over `src` (WordPress lazy-loading stores real URL in `data-src`)
2. Strip query params: `?resize=800,600&ssl=1` → clean URL
3. Rewrite paths: `wp-content/uploads/2024/01/photo.jpg` → `/assets/images/uploads/2024/01/photo.jpg`
4. Download all images locally to `assets/images/uploads/YYYY/MM/`

### Output Summary

| Collection | Count | Format | Naming Convention |
|------------|-------|--------|-------------------|
| `_posts/` | 178 | HTML | `YYYY-MM-DD-slug.html` |
| `_lab/` | 50 | HTML | `slug.html` (no date prefix) |
| `_talks/` | 10 | HTML | `slug.html` |
| `_drafts/` | 8 | HTML | `YYYY-MM-DD-slug.html` |
| `pages/` | 6 | HTML | `slug.html` |

---

## Phase 3 — Content Cleanup

A second Python script (`scripts/clean_lab.py`) tackles WordPress markup remnants that survive extraction. This is the most labor-intensive phase — WordPress themes (especially Visual Composer / WPBakery) inject deeply nested wrapper divs, custom classes, and inline styles.

### Script: `scripts/clean_lab.py`

Run modes:
```bash
python3 scripts/clean_lab.py              # Dry-run: prints what would change
python3 scripts/clean_lab.py --apply      # Apply changes in-place
python3 scripts/clean_lab.py --backup     # Backup to _lab_backup/ then apply
python3 scripts/clean_lab.py --file x.html # Process single file
```

### Cleanup Pipeline (19 steps)

The script uses BeautifulSoup and operates in strict order:

1. **Strip Gutenberg comments** — Remove `<!-- wp:paragraph -->`, `<!-- /wp:image -->`, `<!-- wp:jetpack/slideshow {...} /-->` etc. via regex
2. **Strip malformed attributes** — Remove orphaned `thb_animated_color="..."` text from Visual Composer
3. **Embed bare media URLs** — Convert plain YouTube/Loom URLs on their own line to responsive `<iframe>` embeds wrapped in `<div class="aspect-video">`
4. **Handle inline media URLs** — Convert YouTube URLs mixed with other content
5. **Parse with BeautifulSoup** — Switch from regex to DOM manipulation
6. **Detect Twitter embeds** — Flag `blockquote.twitter-tweet` for widget script injection
7. **Clean WordPress figures** — Unwrap `figure.wp-block-image` to plain `<img>`, preserve `<figcaption>`
8. **Remove decorative elements** — Delete `a.btn-text`, `div.arrow`, `div.thb-divider-container`, `span.vc_sep_holder`, etc.
9. **Unwrap Visual Composer containers** — Multiple passes to peel nested wrappers: `div.wpb_row`, `div.row-fluid`, `div.vc_inner`, `figure.wp-block-embed`, `figure.wp-block-gallery`, etc.
10. **Strip remaining VC/WP wrapper divs** — Regex match against comprehensive class pattern list (handles `vc_custom_*`, `thb-*`, `wp-block-*`, etc.)
11. **Unwrap lightbox anchors** — `<a>` tags that only wrap an `<img>` pointing to the same image
12. **Clean all images** — Strip `data-src`, `srcset`, `sizes`, `width`, `height`, `decoding`, `fetchpriority`, `title`; remove WP classes (`wp-image-*`, `aligncenter`, `size-*`, etc.); add `loading="lazy"`; remove inline styles
13. **Unwrap style-only spans** — `<span style="color:...">` that only wrap text
14. **Strip all inline styles** — Remove `style` attribute from every element
15. **Fix asterisk headings** — `<h2><strong>*Title*</strong></h2>` → `<h2>Title</h2>`
16. **Remove duplicate `<h1>`** — If `<h1>` text matches frontmatter title, remove it (layout renders it)
17. **Remove empty elements** — Iteratively delete `<p>`, `<div>`, `<span>`, `<h1>`–`<h6>` with no text and no meaningful children
18. **Strip WP/VC IDs and classes** — Final pass to remove `thb-*` IDs and `wp-element-*` classes from all elements
19. **Append Twitter widget script** — If Twitter embeds were detected, add `<script async src="https://platform.twitter.com/widgets.js">` at the end

### Final Whitespace Pass

After DOM cleanup, a text-level pass collapses multiple blank lines, trims trailing whitespace, and ensures files end with a single newline.

---

## Phase 4 — Jekyll Architecture

### Directory Structure

```
wordkyll/
├── _config.yml          # Site config, collections, defaults, plugins
├── _posts/              # 178 blog posts (HTML + Markdown)
├── _lab/                # 50 portfolio projects
├── _talks/              # 10 speaking engagements
├── _notes/              # 3 short-form notes (Markdown only)
├── _drafts/             # 8 unpublished posts
├── _layouts/            # 7 templates
│   ├── default.html     # Base: <html>, <head>, fonts, Prism.js, SEO
│   ├── home.html        # Homepage: hero, feature boxes, posts, talks
│   ├── post.html        # Blog: breadcrumbs, growth stage, prose, related
│   ├── portfolio.html   # Lab + Talks: password gate, external links
│   ├── note.html        # Notes: tags, references grid
│   ├── garden.html      # Garden posts: minimal, custom bg
│   └── page.html        # Generic: title + prose content
├── _includes/           # 7 reusable components
│   ├── header.html      # Nav, mobile menu, dark mode toggle
│   ├── footer.html      # Newsletter form, social links, theme toggle
│   ├── post-card.html   # Blog post card (used in grids)
│   ├── post-placeholder.html  # Emoji-based fallback when no image
│   ├── portfolio-card.html    # Project card for lab/talks
│   ├── breadcrumbs.html       # Section breadcrumb navigation
│   └── longform-sidebar.html  # Homepage desktop sidebar
├── _plugins/
│   └── sha256_filter.rb # Custom Liquid filter for password hashing
├── pages/               # 6 standalone pages
├── assets/
│   ├── css/main.css     # Tailwind directives + custom CSS
│   └── images/uploads/  # All media, organized by YYYY/MM/
├── garden/index.html    # Garden page with search JS
├── notes/index.html     # Notes page with tag filter JS
├── scripts/             # Python migration/cleanup scripts
├── Gemfile              # Ruby dependencies
├── package.json         # Node dependencies
├── tailwind.config.js   # Tailwind theme configuration
├── postcss.config.js    # PostCSS pipeline
└── netlify.toml         # Netlify deployment config
```

### Collection Configuration (`_config.yml`)

```yaml
permalink: /:year/:month/:day/:title/   # Preserves WordPress URL structure

collections:
  lab:
    output: true
    permalink: /lab/:title/
  talks:
    output: true
    permalink: /talks/:title/
  notes:
    output: true
    permalink: /notes/:title/

defaults:
  - scope: { path: "", type: "posts" }
    values: { layout: "post" }
  - scope: { path: "", type: "lab" }
    values: { layout: "portfolio" }
  - scope: { path: "", type: "talks" }
    values: { layout: "portfolio" }    # Talks share portfolio layout
  - scope: { path: "", type: "notes" }
    values: { layout: "note" }
  - scope: { path: "pages" }
    values: { layout: "page" }
```

### Plugin Ecosystem

| Plugin | Purpose |
|--------|---------|
| `jekyll-postcss-v2` | PostCSS/Tailwind integration during Jekyll build |
| `jekyll-feed` | RSS/Atom feeds (blog, lab, notes — 3 separate feeds) |
| `jekyll-sitemap` | Auto-generated `sitemap.xml` and `robots.txt` |
| `jekyll-seo-tag` | `<meta>` tags, Open Graph, Twitter Cards, JSON-LD |
| `sha256_filter.rb` | Custom Liquid filter: `{{ password \| sha256 }}` for client-side password gating |

### Layout Hierarchy

```
default.html              ← Base: <html>, <head>, fonts, Prism.js, dark mode init, SEO
├── home.html             ← Homepage: hero section, feature boxes, recent posts, talks
├── post.html             ← Blog: breadcrumbs, growth stage badge, prose, references, related posts
├── portfolio.html        ← Lab + Talks: password protection, external links, Schema.org
├── note.html             ← Notes: tag badges, references grid
├── garden.html           ← Garden posts: minimal layout, custom background
└── page.html             ← Generic: title + prose content block
```

---

## Phase 5 — Design System

### Tailwind Configuration

The design system lives in `tailwind.config.js` with custom tokens for colors, fonts, and typography.

**Font Families:**

| Token | Family | Usage |
|-------|--------|-------|
| `font-serif` | STIX Two Text | Headings, prose body, article content |
| `font-sans` | Noto Sans | Default body text, UI elements |
| `font-nav` | Shantell Sans | Nav labels, section headers, buttons |
| `font-mono` | Intel One Mono | Code blocks, dates, technical labels |

All loaded via Google Fonts CDN with `<link rel="preconnect">` for performance.

**Color System:**

```
Accents:     #6a9a7b (links, CTAs)  →  #5e8e6f (hover)  →  #dd8940 (alt/orange)
Surfaces:    #ebedea (light bg)     →  #2d2d2b (dark bg)  →  #323230 (dark cards)
Text:        #3a3f3b (body)         →  #3a4a40 (headers)
Links:       #5e9a74 (default)      →  #b85f2c (hover)
Semantic:    green/yellow/emerald scales (100–900) for growth stage badges
Gray:        Warm green-tinted gray scale (50–900)
```

**Typography Plugin:**

Custom `prose-green` variant configures link colors, code styling (background, border, border-radius, font), and `pre` block appearance. The `prose-invert` variant overrides for dark mode.

Applied to all article bodies as:
```html
<div class="prose prose-lg prose-green dark:prose-invert max-w-none font-serif
  prose-headings:font-serif prose-headings:font-normal
  prose-a:text-accent prose-a:no-underline hover:prose-a:text-accent-hover
  prose-code:font-mono prose-code:text-sm
  prose-img:ring-1 prose-img:ring-black/10 dark:prose-img:ring-white/10">
```

### Dark Mode

- **Strategy:** `darkMode: 'class'` in Tailwind config — `.dark` class on `<html>`
- **No-flash init:** Inline `<script>` in `<head>` checks `localStorage` then `prefers-color-scheme` before first paint
- **Toggle:** `toggleTheme()` function in `header.html` bound to 3 buttons (desktop, mobile menu, footer)
- **Storage:** `localStorage.setItem('theme', 'dark' | 'light')`
- **Pattern:** Every element uses `class="light-value dark:dark-value"`

```javascript
// Inline in <head> — runs before body render
(function() {
  var saved = localStorage.getItem('theme');
  if (saved === 'dark' || (!saved && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
    document.documentElement.classList.add('dark');
  }
})();
```

### Custom CSS (`assets/css/main.css`)

Beyond Tailwind utilities, custom CSS handles:

- **Prism.js code blocks:** Overrides for `pre[class*="language-"]` with `#2d2d2d` background (light) and `#1a1a1a` (dark), plus zebra-stripe line backgrounds via `repeating-linear-gradient`
- **Post placeholders:** `linear-gradient` backgrounds using a CSS custom property `--ph-hue` derived from title length, with separate gradients for light/dark and growth stages
- **Now page timeline:** Vertical line with `::before` pseudo-element, dot markers on `h2::before`, responsive padding
- **Scrollbar hiding:** `.scrollbar-hide` utility for horizontal scroll containers

### Responsive Strategy

Mobile-first using Tailwind default breakpoints:

```
sm: 640px  — Tablets
md: 768px  — Medium tablets
lg: 1024px — Laptops
xl: 1280px — Desktops
```

Common patterns:
```html
<!-- Container -->
<div class="max-w-6xl mx-auto px-4 sm:px-6">

<!-- Grid -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">

<!-- Type scale -->
<h1 class="text-3xl sm:text-4xl lg:text-5xl">

<!-- Nav visibility -->
<nav class="hidden sm:flex">  <!-- Desktop -->
<div class="sm:hidden">       <!-- Mobile -->
```

---

## Phase 6 — Feature Layer

### Digital Garden (Growth Stages)

Posts can declare a `growth_stage` in frontmatter: `seedling`, `blossoming`, or `flourishing`. The post layout renders a colored badge with emoji:

| Stage | Emoji | Badge Colors |
|-------|-------|-------------|
| seedling | &#127793; | Green background (`bg-green-100 text-green-800`) |
| blossoming | &#127804; | Yellow background (`bg-yellow-100 text-yellow-800`) |
| flourishing | &#127795; | Emerald background (`bg-emerald-100 text-emerald-800`) |

The garden index page (`garden/index.html`) provides client-side search filtering by title, description, and content snippets, with load-more pagination (25 posts per page).

### Password Protection

16 lab projects use `protected: true` in frontmatter. The protection mechanism:

1. `_config.yml` stores the plaintext password (`portfolio_password`)
2. At build time, `_plugins/sha256_filter.rb` hashes it: `{{ site.portfolio_password | sha256 }}`
3. The hash is embedded in the page's `<script>` block
4. Content lives inside a `<template id="protected-content">` (not rendered by default)
5. On form submit, the entered password is hashed client-side via `crypto.subtle.digest('SHA-256', ...)`
6. If hashes match, content is cloned from the `<template>` into the DOM
7. `sessionStorage` persists the unlock for the browser session

**Limitation:** The hash is visible in page source. This is adequate for casual gating, not real security.

### Post Placeholders

When a post has no featured image, an emoji-based placeholder is generated:

- A hue value is derived from the post title length: `--ph-hue: {{ title.size | modulo: 360 }}`
- CSS `linear-gradient` uses this hue for a subtle, unique background per post
- Different gradients for light mode, dark mode, and blossoming stage
- Defined in `_includes/post-placeholder.html` and `assets/css/main.css`

### References System

Posts and notes can declare structured references in frontmatter:

```yaml
references:
  - title: "Resource Name"
    url: "https://example.com"
    description: "Why it's relevant"
```

Rendered as a card grid in `post.html` and `note.html` with bookmark icons, hover states, and external link targets.

### Intended Audience

Posts can declare `intended_audience` in frontmatter, rendered as a callout component above the article body:

```yaml
intended_audience: "Engineers and product managers working on AI-powered tools"
```

### Notes Tag Filtering

The notes index page (`notes/index.html`) supports tag-based filtering with URL hash sync — clicking a tag updates the hash (`#typography`), and loading the page with a hash pre-selects that filter.

---

## Phase 7 — Build & Deployment

### Build Pipeline

```
Source files → Jekyll 4.4 → PostCSS (Tailwind + Autoprefixer) → _site/
```

PostCSS config (`postcss.config.js`):
```javascript
module.exports = {
  plugins: [
    require('tailwindcss'),
    require('autoprefixer'),
  ]
}
```

**Important:** cssnano is intentionally excluded from the pipeline due to a csso/css-tree incompatibility. Do not re-add it.

### Netlify Configuration (`netlify.toml`)

```toml
[build]
  command = "bundle exec jekyll build"
  publish = "_site"

[build.environment]
  JEKYLL_ENV = "production"
  RUBY_VERSION = "3.2.0"
  NODE_VERSION = "18"
```

### Redirects

Preserve WordPress URL compatibility and handle renamed pages:

```toml
/connect-with-andrew/* → /connect/   (301)
/whoami/now/           → /now/        (301)
/feed/                 → /feed.xml    (301)
/*                     → /404.html    (404)
```

### RSS Feeds

Three separate feeds via `jekyll-feed`:

| Feed | Path | Content |
|------|------|---------|
| Blog | `/feed.xml` | Latest 20 posts |
| Lab | `/lab/feed.xml` | Portfolio projects |
| Notes | `/notes/feed.xml` | Notes |

### Generated Outputs

| File | Generator | Purpose |
|------|-----------|---------|
| `/sitemap.xml` | jekyll-sitemap | XML sitemap for search engines |
| `/robots.txt` | jekyll-sitemap | Crawl directives, references sitemap |
| `/feed.xml` | jekyll-feed | Blog RSS (20 items) |
| `/lab/feed.xml` | jekyll-feed | Portfolio RSS |
| `/notes/feed.xml` | jekyll-feed | Notes RSS |

---

## Phase 8 — Refinement

### Responsiveness Audit

- **Viewport overflow:** `min-w-full` on fixed/absolute elements causes horizontal scroll on mobile — replaced with `w-full`
- **Mobile menu:** Added `overflow-hidden` on `<body>` when open, `divide-y` navigation, compressed spacing
- **Touch targets:** Ensured all interactive elements meet 44px minimum touch target size
- **Image containment:** `prose-img:w-full prose-img:max-w-full prose-img:h-auto` prevents images from overflowing containers

### Dark Mode Code Blocks

Prism.js doesn't wrap individual lines in elements, making per-line styling impossible via selectors. Solution: `repeating-linear-gradient` on `pre` background creates zebra-stripe effect:

```css
.prose pre {
  background-image: repeating-linear-gradient(
    180deg,
    transparent, transparent 1.4rem,
    rgba(0, 0, 0, 0.04) 1.4rem, rgba(0, 0, 0, 0.04) 2.8rem
  ) !important;
  background-position: 0 1.25em;
  background-size: 100% 2.8rem;
}
```

Dark mode uses `rgba(255, 255, 255, 0.03)` and a darker base (`#1a1a1a`) with an inset box-shadow border.

### SEO

- Meta descriptions on all pages via `{% seo %}` plugin + `description` frontmatter
- Open Graph and Twitter Card tags auto-generated
- Fallback OG image when no `page.image` is set
- `article:tag` and `article:section` meta tags from frontmatter
- Keywords meta tag from post tags
- Schema.org JSON-LD on portfolio pages

### Content-to-Markdown Conversion

Select posts converted from legacy HTML to Markdown for easier editing. During conversion:
- Inline references moved to `references` frontmatter array
- `intended_audience` moved from content to frontmatter field
- WordPress Gutenberg block comments stripped

---

## Lessons Learned & Pitfalls

### Build & Tooling

1. **cssnano breaks with csso/css-tree** — Installing cssnano pulls in csso which has an incompatibility with certain css-tree versions. CSS minification had to be dropped from the PostCSS pipeline. The site ships unminified CSS.

2. **Tailwind arbitrary `calc()` values fail with spaces** — `w-[calc(100%-2rem)]` works but `w-[calc(100% - 2rem)]` does not compile. Remove spaces inside `calc()` in arbitrary Tailwind values.

3. **`jekyll-postcss-v2` requires empty frontmatter** — The CSS file (`assets/css/main.css`) must start with empty YAML frontmatter delimiters (`---\n---`) for Jekyll to process it through PostCSS. Without this, Tailwind directives pass through unprocessed.

### WordPress Content

4. **Visual Composer nesting is extreme** — A single image can be wrapped in 5+ layers of `div.wpb_row > div.row-fluid > div.vc_inner > div.thb-image-inner > figure > a > img`. The cleanup script needs multiple unwrapping passes (up to 5 iterations).

5. **WordPress artifacts persist in unexpected places** — Social share buttons, portfolio navigation markup, Gutenberg comments, and VC decorative elements (`div.arrow`, `div.thb-divider-container`) survive initial extraction and must be explicitly targeted.

6. **`data-src` vs `src` for lazy-loaded images** — WordPress lazy-loading plugins store the real image URL in `data-src` and put a placeholder in `src`. Always check `data-src` first during extraction.

7. **Image query parameters must be stripped** — WordPress appends `?resize=800,600&ssl=1` to image URLs. These break when served statically and must be removed during extraction.

8. **Gutenberg block comments have varied syntax** — Some are self-closing (`<!-- wp:jetpack/slideshow {...} /-->`), some have content between open/close tags. The regex must handle both: `<!--\s*/?wp:.*?-->`.

### Design & CSS

9. **Inline `style` attributes override Tailwind dark mode** — WordPress content often has inline `style="color: #333"` on elements. These override `dark:text-cream` classes because inline styles have higher specificity. Must strip all inline styles during cleanup.

10. **Prism.js line styling requires background gradients** — Prism doesn't wrap lines in elements, so you can't use `:nth-child` for zebra striping. Use `repeating-linear-gradient` on the `pre` element's background-image, carefully calibrated to line-height.

11. **`min-w-full` on positioned elements causes mobile overflow** — Fixed or absolute elements with `min-w-full` extend beyond the viewport. Use `w-full` instead, and add `overflow-x-hidden` on `<body>` as a safety net.

### Architecture

12. **Client-side password protection is transparent** — The SHA-256 hash is visible in page source. Anyone can extract it and compute the password (or just read the `<template>` content). This is by design — adequate for "please don't casually browse this" gating, not real access control.

13. **Multiple collections can share a layout** — `_lab/` and `_talks/` both use `portfolio.html` via `_config.yml` defaults. This works well when the content structure is similar, avoiding layout duplication.

14. **Preserve WordPress permalink structure** — Setting `permalink: /:year/:month/:day/:title/` in `_config.yml` ensures all existing URLs continue working. Combined with Netlify redirects for renamed pages, this prevents 404s from external links.

---

## File Reference

Quick-reference for all project files and when you'd touch them.

### Configuration

| File | Purpose | When to Touch |
|------|---------|---------------|
| `_config.yml` | Site config, collections, defaults, plugins, feed settings | Adding collections, changing permalinks, updating plugins |
| `Gemfile` | Ruby dependencies | Adding Jekyll plugins |
| `package.json` | Node dependencies, build scripts | Adding PostCSS plugins, updating Tailwind |
| `tailwind.config.js` | Colors, fonts, dark mode, typography plugin | Changing design tokens, adding content paths |
| `postcss.config.js` | PostCSS plugin pipeline | Adding/removing PostCSS plugins (do NOT add cssnano) |
| `netlify.toml` | Build command, env vars, redirects | Adding redirects, changing build settings |

### Layouts

| File | Extends | Used By |
|------|---------|---------|
| `_layouts/default.html` | — | All other layouts |
| `_layouts/home.html` | default | Homepage (`index.html`) |
| `_layouts/post.html` | default | `_posts/` |
| `_layouts/portfolio.html` | default | `_lab/`, `_talks/` |
| `_layouts/note.html` | default | `_notes/` |
| `_layouts/garden.html` | default | Garden-tagged posts |
| `_layouts/page.html` | default | `pages/` |

### Includes

| File | Used In | Parameters |
|------|---------|------------|
| `_includes/header.html` | `default.html` | — |
| `_includes/footer.html` | `default.html` | — |
| `_includes/post-card.html` | `home.html`, `post.html` | `post` (object) |
| `_includes/post-placeholder.html` | `post-card.html`, layouts | `post`, `class`, `emoji_size` |
| `_includes/portfolio-card.html` | `home.html`, `experiments.html` | `project` (object) |
| `_includes/breadcrumbs.html` | `post.html`, `portfolio.html`, `note.html` | `section`, `section_url` |
| `_includes/longform-sidebar.html` | `home.html` | — |

### Content Collections

| Directory | Count | Format | Layout |
|-----------|-------|--------|--------|
| `_posts/` | 178 | HTML + Markdown | post |
| `_lab/` | 50 | HTML + Markdown | portfolio |
| `_talks/` | 10 | HTML | portfolio |
| `_notes/` | 3 | Markdown | note |
| `_drafts/` | 8 | HTML | post |
| `pages/` | 6 | HTML + Markdown | page/default |

### Scripts

| File | Purpose |
|------|---------|
| `scripts/clean_lab.py` | WordPress markup cleanup (BeautifulSoup-based, 19-step pipeline) |

### Other

| File | Purpose |
|------|---------|
| `_plugins/sha256_filter.rb` | Custom Liquid filter for password hashing |
| `assets/css/main.css` | Tailwind directives + custom CSS (Prism, placeholders, timeline) |
| `garden/index.html` | Garden page with client-side search and pagination |
| `notes/index.html` | Notes page with tag filtering and URL hash sync |
| `CLAUDE.md` | Engineering directive for AI-assisted development |
