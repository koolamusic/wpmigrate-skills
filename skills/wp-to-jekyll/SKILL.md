---
name: wp-to-jekyll
description: Migrate WordPress content to Jekyll. Use when asked to "convert WordPress to Jekyll", "migrate WP to Jekyll", "set up Jekyll from WordPress", "WordPress to static site", or "export WordPress to markdown". Covers content extraction, format conversion, Jekyll architecture setup, and deployment.
argument-hint: <wordpress-source-path-or-question>
---

# WordPress to Jekyll Migration

Step-by-step guide for converting WordPress content into a Jekyll static site. Covers extracting content from a WordPress HTML clone or XML export, transforming it into Jekyll-compatible files with proper frontmatter, setting up collections, and deploying.

## When to Apply

Reference this skill when:
- Converting a WordPress site to Jekyll
- Setting up a Jekyll project from WordPress content
- Transforming WordPress HTML/XML into Jekyll markdown or HTML with frontmatter
- Cleaning up WordPress markup artifacts for static site use
- Configuring Jekyll collections to match WordPress content types

## Prerequisites

### System Dependencies

| Tool | Version | Purpose |
|------|---------|---------|
| Ruby | 3.2+ | Jekyll runtime |
| Bundler | latest | Ruby dependency management |
| Node.js | 18+ | Tailwind CSS / asset compilation |
| Python 3 | 3.8+ | Content extraction and cleanup scripts |
| BeautifulSoup4 | latest | HTML parsing in Python scripts |

### Gemfile

```ruby
gem 'jekyll', '~> 4.4'
gem 'webrick'               # Dev server (Ruby 3.x dropped it from stdlib)
gem 'jekyll-postcss-v2'     # PostCSS/Tailwind integration (optional)
gem 'jekyll-feed'           # RSS/Atom feed generation
gem 'jekyll-sitemap'        # XML sitemap
gem 'jekyll-seo-tag'        # Meta tags and structured data
gem 'logger'                # Ruby 3.x stdlib extraction
gem 'csv'                   # Ruby 3.x stdlib extraction
gem 'base64'                # Ruby 3.x stdlib extraction
```

### Bootstrap

```bash
bundle install && npm install   # Install all dependencies
bundle exec jekyll serve        # Dev server at localhost:4000
bundle exec jekyll build        # Production build to _site/
```

## Phase 1 — Source Acquisition

You need WordPress content in one of two forms:

### Option A: Site Mirror (HTML clone)

Mirror the live WordPress site to capture rendered HTML and media:

```bash
# HTTrack
httrack "https://your-site.com" -O ./mirror \
  --mirror --robots=0 --depth=10

# wget alternative
wget --mirror --convert-links --adjust-extension \
  --page-requisites --no-parent https://your-site.com
```

The mirror gives you the rendered output of every page, including page builder content, plugin output, and all media files.

### Option B: WordPress XML Export (WXR)

Export from WP Admin → Tools → Export. This gives you structured content with metadata but no media files and no rendered page builder output.

### Recommended: Use Both

The XML export provides metadata (dates, categories, tags). The mirror provides clean rendered HTML and media files. Cross-reference both for the best result.

## Phase 2 — Content Extraction

A Python script parses each source file, classifies it by URL pattern, extracts frontmatter, pulls body content, and writes Jekyll-compatible files.

### URL-to-Collection Mapping

| WordPress URL Pattern | Jekyll Output | Collection |
|----------------------|---------------|------------|
| `/YYYY/MM/DD/slug/` | `_posts/YYYY-MM-DD-slug.html` | Blog posts |
| `/category/slug/` | Skip (Jekyll generates these) | — |
| `/tag/slug/` | Skip (Jekyll generates these) | — |
| `/author/slug/` | Skip or `pages/` | — |
| `/page-slug/` | `pages/slug.html` | Standalone pages |
| Custom post types | `_collection-name/slug.html` | Custom collections |

### Frontmatter Extraction

Derive YAML frontmatter from WordPress HTML meta tags or WXR XML:

**From HTML mirror (OpenGraph tags):**
```python
# Meta tags → frontmatter fields
'og:title'               → title
'og:description'         → description
'og:image'               → image
'og:url'                 → permalink
'article:published_time' → date
'article:modified_time'  → last_modified_at
'article:tag'            → tags (multiple)
'article:section'        → categories
```

**From WXR XML:**
```python
# XML elements → frontmatter fields
'<title>'                → title
'<wp:post_date>'         → date
'<category domain="category">' → categories
'<category domain="post_tag">' → tags
'<wp:status>'            → published (true/false)
'<content:encoded>'      → body content
```

### Content Body Extraction (from HTML mirror)

Isolate the article body from WordPress page chrome:

```python
from bs4 import BeautifulSoup

soup = BeautifulSoup(html, 'html.parser')
# Target the main content div — class varies by theme
content = (
    soup.find("div", class_="post-content") or
    soup.find("div", class_="entry-content") or
    soup.find("article", class_="post") or
    soup.find("main")
)
```

This strips navigation, sidebars, related posts, comments, and footer.

### Image Handling

1. Prefer `data-src` over `src` (WordPress lazy-loading stores real URL in `data-src`)
2. Strip query params: `?resize=800,600&ssl=1` → clean URL
3. Rewrite paths: `wp-content/uploads/2024/01/photo.jpg` → `/assets/images/uploads/2024/01/photo.jpg`
4. Download all images locally to `assets/images/uploads/YYYY/MM/`

### Example Output

```
_posts/2024-01-15-my-blog-post.html
```

```yaml
---
layout: post
title: "My Blog Post Title"
date: 2024-01-15
description: "A brief description from the meta tag"
image: /assets/images/uploads/2024/01/featured.jpg
categories: [technology, web-development]
tags: [jekyll, wordpress, migration]
---
<p>The extracted and cleaned article content goes here...</p>
```

## Phase 3 — Content Cleanup

WordPress themes inject deeply nested wrapper divs, custom classes, and inline styles. A cleanup script handles these systematically.

### Cleanup Pipeline

Run a Python script with BeautifulSoup to clean WordPress artifacts:

1. **Strip Gutenberg comments** — Remove `<!-- wp:paragraph -->`, `<!-- /wp:image -->`, etc.
   ```python
   import re
   content = re.sub(r'<!--\s*/?wp:.*?-->', '', content)
   ```

2. **Unwrap page builder containers** — Peel nested wrappers from Visual Composer, Elementor, Divi:
   ```python
   # Classes to unwrap (element replaced by its children)
   unwrap_classes = [
       'wpb_row', 'row-fluid', 'vc_inner', 'vc_column_container',
       'wp-block-image', 'wp-block-embed', 'wp-block-gallery',
       'elementor-widget-container', 'et_pb_module_inner'
   ]
   for cls in unwrap_classes:
       for el in soup.find_all(class_=cls):
           el.unwrap()
   ```

3. **Clean images** — Strip WordPress-specific attributes, add lazy loading:
   ```python
   for img in soup.find_all('img'):
       # Use data-src if available (lazy loading plugins)
       if img.get('data-src'):
           img['src'] = img['data-src']
       # Strip WP attributes
       for attr in ['data-src', 'srcset', 'sizes', 'width', 'height',
                     'decoding', 'fetchpriority', 'title']:
           img.attrs.pop(attr, None)
       # Strip WP classes
       if img.get('class'):
           img['class'] = [c for c in img['class']
                          if not c.startswith(('wp-image-', 'aligncenter', 'size-'))]
       img['loading'] = 'lazy'
   ```

4. **Strip inline styles** — WordPress content often has inline `style` attributes that override new theme CSS:
   ```python
   for el in soup.find_all(style=True):
       del el['style']
   ```

5. **Remove empty elements** — Iteratively delete empty `<p>`, `<div>`, `<span>`, headings:
   ```python
   for tag_name in ['p', 'div', 'span', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6']:
       for el in soup.find_all(tag_name):
           if not el.get_text(strip=True) and not el.find(['img', 'iframe', 'video']):
               el.decompose()
   ```

6. **Remove duplicate headings** — If `<h1>` text matches the frontmatter title, remove it (the Jekyll layout renders it)

7. **Embed media URLs** — Convert bare YouTube/Vimeo URLs to responsive iframes:
   ```python
   # Convert: https://www.youtube.com/watch?v=XXXX
   # To: <div class="aspect-video"><iframe src="https://www.youtube.com/embed/XXXX" ...></iframe></div>
   ```

### Running Cleanup

```bash
python3 scripts/clean.py              # Dry-run: prints changes
python3 scripts/clean.py --apply      # Apply changes in-place
python3 scripts/clean.py --backup     # Backup originals first
python3 scripts/clean.py --file x.html # Process single file
```

## Phase 4 — Jekyll Architecture

### Directory Structure

```
your-jekyll-site/
├── _config.yml           # Site config, collections, defaults, plugins
├── _posts/               # Blog posts (YYYY-MM-DD-slug.html or .md)
├── _drafts/              # Unpublished posts
├── _layouts/             # Page templates
│   ├── default.html      # Base layout: <html>, <head>, nav, footer
│   ├── post.html         # Blog post template
│   └── page.html         # Generic page template
├── _includes/            # Reusable components
│   ├── header.html
│   ├── footer.html
│   └── post-card.html
├── pages/                # Standalone pages (about, contact, etc.)
├── assets/
│   ├── css/
│   └── images/uploads/   # Migrated WordPress media (YYYY/MM/)
├── Gemfile               # Ruby dependencies
├── package.json          # Node dependencies (if using Tailwind)
└── netlify.toml          # Deployment config (optional)
```

### Custom Collections

Map WordPress custom post types to Jekyll collections in `_config.yml`:

```yaml
# Preserve WordPress URL structure
permalink: /:year/:month/:day/:title/

collections:
  # Example: WordPress 'portfolio' post type → Jekyll collection
  portfolio:
    output: true
    permalink: /portfolio/:title/
  # Example: WordPress 'talks' post type
  talks:
    output: true
    permalink: /talks/:title/

# Set default layouts per collection
defaults:
  - scope: { path: "", type: "posts" }
    values: { layout: "post" }
  - scope: { path: "", type: "portfolio" }
    values: { layout: "portfolio" }
  - scope: { path: "", type: "talks" }
    values: { layout: "portfolio" }
  - scope: { path: "pages" }
    values: { layout: "page" }
```

### Key Config Settings

```yaml
# _config.yml
title: "Your Site Name"
url: "https://your-site.com"
description: "Site description for SEO"

# Plugins
plugins:
  - jekyll-feed
  - jekyll-sitemap
  - jekyll-seo-tag
  - jekyll-postcss-v2    # Only if using Tailwind

# Feed configuration (generates RSS)
feed:
  collections:
    - posts
    - portfolio
```

## Phase 5 — Content Format Conversion

### HTML to Markdown (Optional)

You can optionally convert extracted HTML content to Markdown for easier editing. Not all content converts cleanly — complex layouts, tables, and embedded media may be better left as HTML.

**Good candidates for Markdown conversion:**
- Text-heavy blog posts
- Simple pages with headings, paragraphs, lists, links, images

**Keep as HTML:**
- Posts with complex layouts or embedded widgets
- Content with custom CSS classes you want to preserve
- Pages with embedded iframes, forms, or interactive elements

**Conversion approach:**
```python
import markdownify

# Convert HTML to Markdown, preserving images and links
markdown_content = markdownify.markdownify(
    html_content,
    heading_style="atx",        # Use # style headings
    bullets="-",                 # Use - for unordered lists
    strip=['script', 'style']   # Remove script and style tags
)
```

### Frontmatter Field Mapping

| WordPress Field | Jekyll Frontmatter | Notes |
|----------------|-------------------|-------|
| Post title | `title` | Wrap in quotes if contains colons |
| Published date | `date` | Format: `YYYY-MM-DD` or `YYYY-MM-DD HH:MM:SS +0000` |
| Slug | `permalink` | Only if overriding the default pattern |
| Categories | `categories` | Array: `[cat1, cat2]` |
| Tags | `tags` | Array: `[tag1, tag2]` |
| Featured image | `image` | Path to local file in `assets/images/` |
| Meta description | `description` | From Yoast/RankMath or og:description |
| Author | `author` | String or reference to `_data/authors.yml` |
| Post status: draft | Move to `_drafts/` | Drafts don't need a date prefix in filename |
| Custom fields | Custom frontmatter keys | Map ACF fields to meaningful frontmatter names |
| Password protected | `protected: true` | Implement client-side gating in layout |

## Phase 6 — Build & Deployment

### Local Development

```bash
bundle exec jekyll serve --livereload    # Dev server with auto-reload
bundle exec jekyll serve --drafts        # Include drafts
bundle exec jekyll build                 # Production build to _site/
```

### Netlify Deployment

```toml
# netlify.toml
[build]
  command = "bundle exec jekyll build"
  publish = "_site"

[build.environment]
  JEKYLL_ENV = "production"
  RUBY_VERSION = "3.2.0"
  NODE_VERSION = "18"
```

### GitHub Pages Deployment

```yaml
# .github/workflows/jekyll.yml
name: Deploy Jekyll
on:
  push:
    branches: [main]
jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
      - run: bundle exec jekyll build
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./_site
```

### URL Redirects

Preserve WordPress URLs that changed during migration:

```toml
# netlify.toml redirects
[[redirects]]
  from = "/feed/"
  to = "/feed.xml"
  status = 301

[[redirects]]
  from = "/wp-content/uploads/*"
  to = "/assets/images/uploads/:splat"
  status = 301
```

For Jekyll-native redirects, use the `jekyll-redirect-from` gem:
```yaml
# In a post's frontmatter
redirect_from:
  - /old-url/
  - /another-old-url/
```

## Lessons Learned & Pitfalls

### Content Extraction

1. **WordPress lazy loading: `data-src` vs `src`** — Lazy-loading plugins store the real image URL in `data-src` and a placeholder in `src`. Always check `data-src` first.

2. **Image query parameters must be stripped** — WordPress appends `?resize=800,600&ssl=1`. These break on static hosting.

3. **Gutenberg comments have varied syntax** — Some are self-closing (`<!-- wp:jetpack/slideshow {...} /-->`), some wrap content. Use regex: `<!--\s*/?wp:.*?-->`.

4. **Visual Composer nesting is extreme** — A single image can be wrapped in 5+ layers of divs. Your cleanup script needs multiple unwrapping passes.

### Build & Tooling

5. **cssnano + csso/css-tree incompatibility** — If using PostCSS, do NOT add cssnano. It pulls in csso which breaks with certain css-tree versions.

6. **`jekyll-postcss-v2` requires empty frontmatter** — Your CSS file must start with `---\n---` for Jekyll to process it through PostCSS.

7. **Tailwind arbitrary `calc()` values fail with spaces** — `w-[calc(100%-2rem)]` works; `w-[calc(100% - 2rem)]` does not.

### Design & CSS

8. **Inline `style` attributes override Tailwind dark mode** — WordPress content with `style="color: #333"` overrides `dark:text-white`. Strip all inline styles during cleanup.

9. **Multiple collections can share a layout** — Use `_config.yml` defaults to assign the same layout to similar collection types, avoiding duplication.

10. **Preserve WordPress permalink structure** — Set `permalink: /:year/:month/:day/:title/` to maintain existing URLs and prevent 404s from external links and search engines.
