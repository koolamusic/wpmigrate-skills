---
name: wordpress-migration-best-practices
description: Best practices for migrating content out of WordPress. Use when asked to "migrate from WordPress", "export WordPress content", "move off WordPress", "WordPress migration strategy", or "extract WordPress data". Covers XML export, site mirroring, plugin-specific content, and migration planning.
argument-hint: <target-platform-or-question>
---

# WordPress Migration Best Practices

Comprehensive guide for extracting and migrating content from WordPress sites. Covers multiple extraction strategies, their trade-offs, and best practices for handling WordPress-specific content like custom plugins, WooCommerce, forms, and media.

## When to Apply

Reference these guidelines when:
- Planning a migration away from WordPress
- Exporting content from WordPress for any target platform
- Dealing with complex WordPress sites (WooCommerce, custom plugins, page builders)
- Evaluating which extraction method to use
- Handling media, images, and uploads during migration

## Extraction Strategies

### Strategy 1: WordPress XML Export (WXR)

The built-in WordPress export tool. Go to **WP Admin → Tools → Export** to generate an XML file containing posts, pages, comments, categories, tags, and custom post types.

**What you get:**
- All posts and pages with full content (HTML)
- Categories, tags, and custom taxonomies
- Comments and comment metadata
- Custom post types and their content
- Author information
- Featured image references (URLs, not files)
- Custom field / post meta data

**What you don't get:**
- Actual media files (only URLs/references)
- Theme settings and customizations
- Widget configurations
- Plugin data (WooCommerce products, form entries, etc.)
- Menu structures
- Site options and settings
- Page builder layouts (stored as shortcodes or serialized data in post content)

**Best for:** Simple blogs and content-focused sites with standard posts/pages.

**Usage:**
```bash
# Parse the WXR XML export with a script
python3 scripts/parse-wxr.py wordpress-export.xml --output ./content/

# Or use wp-cli if you have server access
wp export --dir=./exports/ --post_type=post,page
```

**Key considerations:**
- Export in smaller batches if the site has 1000+ posts (the export can time out)
- Custom fields are included but may contain serialized PHP data that needs parsing
- Gutenberg block content is stored as HTML comments within the post content
- Shortcodes from plugins will appear as raw `[shortcode]` text unless the plugin is active

### Strategy 2: Site Mirroring (HTTrack / wget)

Clone the entire rendered site as static HTML files. This captures exactly what visitors see, including all rendered plugin output, theme styling, and dynamic content.

**What you get:**
- Complete rendered HTML of every public page
- All publicly accessible media files (images, PDFs, etc.)
- CSS and JavaScript files
- The exact visual output of the site

**What you don't get:**
- Content structure (no frontmatter, no metadata separation)
- Draft or private content
- Admin-only pages
- Dynamic functionality (forms, search, e-commerce carts)
- Server-side logic (PHP functions, plugin behavior)
- Database content not rendered on public pages

**Best for:** Sites with heavy theme customization or page builder content where the rendered output is more reliable than the raw database content.

**Usage:**
```bash
# HTTrack — full mirror
httrack "https://example.com" -O ./mirror \
  --mirror --robots=0 --depth=10

# wget — alternative approach
wget --mirror --convert-links --adjust-extension \
  --page-requisites --no-parent https://example.com
```

**Key considerations:**
- Password-protected pages won't be captured (server-side auth)
- JavaScript-rendered content may not be captured (use a headless browser for SPAs)
- Look for duplicate pages: `/embed/` variants, print versions, AMP pages
- Check `robots.txt` — some important pages may be blocked from crawlers
- Media files in `wp-content/uploads/` are organized by `YYYY/MM/` — preserve this structure

### Strategy 3: Combined Approach (Recommended)

Use both XML export AND site mirroring together. The XML export gives you structured content with metadata; the mirror gives you rendered output and media files.

**Workflow:**
1. Export XML from WordPress admin for structured content and metadata
2. Mirror the site with HTTrack/wget for rendered HTML and all media
3. Cross-reference: use XML metadata (dates, tags, categories) with mirror content (clean rendered HTML)
4. Use the mirror's media files as the authoritative source for images and uploads

### Strategy 4: Direct Database Access

If you have server/hosting access, query the WordPress MySQL database directly.

**Key tables:**
| Table | Content |
|-------|---------|
| `wp_posts` | All content (posts, pages, revisions, attachments) |
| `wp_postmeta` | Custom fields, featured images, plugin data |
| `wp_terms` / `wp_term_taxonomy` | Categories, tags, custom taxonomies |
| `wp_comments` | Comments with metadata |
| `wp_options` | Site settings, widget configs, plugin settings |
| `wp_usermeta` | User profile data |

**Usage:**
```sql
-- Export all published posts
SELECT ID, post_title, post_content, post_date, post_name, post_type
FROM wp_posts
WHERE post_status = 'publish'
AND post_type IN ('post', 'page')
ORDER BY post_date DESC;

-- Get post metadata (featured images, custom fields)
SELECT p.post_title, pm.meta_key, pm.meta_value
FROM wp_posts p
JOIN wp_postmeta pm ON p.ID = pm.post_id
WHERE p.post_status = 'publish';
```

**Best for:** Large sites where the XML export times out, or when you need access to plugin-specific database tables.

## Handling Plugin-Specific Content

### WooCommerce

WooCommerce stores products, orders, and customer data in custom post types and meta tables.

**Content to extract:**
- Products: `wp_posts` where `post_type = 'product'`
- Product meta: price, SKU, stock, dimensions in `wp_postmeta`
- Product categories: custom taxonomy `product_cat`
- Product images: gallery images stored as serialized arrays in `_product_image_gallery` meta
- Variations: `post_type = 'product_variation'`

**Not typically migrated:**
- Orders and order history (platform-specific)
- Customer accounts (usually recreated on new platform)
- Coupons and promotions (recreated manually)

**Best practice:** Export products to CSV using WooCommerce's built-in exporter (WP Admin → Products → Export), then transform to the target platform's format.

### Contact Forms (Contact Form 7, Gravity Forms, WPForms)

Form plugins store form definitions and submissions differently:

- **Contact Form 7:** Forms stored as `wpcf7_contact_form` post type. Submissions are emailed, not stored in DB by default (unless using Flamingo plugin).
- **Gravity Forms:** Forms in `wp_gf_form` table, entries in `wp_gf_entry`. Export entries from Forms → Import/Export.
- **WPForms:** Forms stored as `wpforms` post type (serialized data). Entries in `wp_wpforms_entries`.

**Best practice:** Export form submissions as CSV. Recreate form structures manually on the new platform — form definitions rarely migrate cleanly between systems.

### Page Builders (Elementor, WPBakery/Visual Composer, Divi)

Page builder content is the hardest to migrate because it's stored as:
- Shortcodes in post content (WPBakery): `[vc_row][vc_column][vc_column_text]Content[/vc_column_text][/vc_column][/vc_row]`
- Serialized JSON in postmeta (Elementor): `_elementor_data` field
- Shortcodes with custom syntax (Divi): `[et_pb_section][et_pb_row]...`

**Best practice:** Use site mirroring to capture the rendered output. The raw shortcode/JSON data is only useful if migrating to another WordPress site with the same page builder installed.

**Cleanup required (from mirrored HTML):**
- Strip nested wrapper divs (5+ levels deep for Visual Composer)
- Remove decorative elements and spacer divs
- Strip inline styles that override new theme styling
- Convert page builder grid layouts to semantic HTML or your new CSS framework's grid

### SEO Plugins (Yoast, Rank Math, All in One SEO)

SEO plugins store metadata in `wp_postmeta`:

| Meta Key (Yoast) | Meta Key (Rank Math) | Content |
|-------------------|---------------------|---------|
| `_yoast_wpseo_title` | `rank_math_title` | Custom page title |
| `_yoast_wpseo_metadesc` | `rank_math_description` | Meta description |
| `_yoast_wpseo_focuskw` | `rank_math_focus_keyword` | Focus keyword |
| `_yoast_wpseo_canonical` | `rank_math_canonical_url` | Canonical URL |

**Best practice:** Export SEO metadata early in the migration. Losing meta descriptions and custom titles impacts search rankings. Map these to your new platform's SEO fields.

### Custom Post Types and Advanced Custom Fields (ACF)

- Custom post types appear in `wp_posts` with their registered `post_type` slug
- ACF fields are stored in `wp_postmeta` with field names as `meta_key`
- ACF field group definitions are stored as `acf-field-group` post type
- Repeater fields use numbered meta keys: `field_name_0_subfield`, `field_name_1_subfield`

**Best practice:** Map ACF fields to your new platform's equivalent (frontmatter fields, structured content types, etc.). Document the field mapping before starting migration.

## Media Migration Best Practices

### Image Handling

1. **Download all media locally** — Don't rely on hotlinking to the old WordPress URLs
2. **Preserve directory structure** — WordPress organizes uploads by `YYYY/MM/`. Keep this or create a clear mapping
3. **Strip query parameters** — WordPress appends `?resize=800,600&ssl=1` to image URLs. Remove these for static hosting
4. **Handle lazy loading** — WordPress lazy-loading plugins store real URLs in `data-src`, not `src`. Always check `data-src` first
5. **Check for CDN URLs** — If the site uses a CDN (Cloudflare, Jetpack Photon), images may be served from a different domain. Map these back to original paths
6. **Regenerate responsive sizes** — WordPress generates multiple sizes per image (`-150x150`, `-300x200`, etc.). Decide whether to keep these or regenerate for your new platform

### Media Inventory Script

```bash
# Find all media references in exported content
grep -roh 'wp-content/uploads/[^"'"'"' ]*' content/ | sort -u > media-inventory.txt

# Download all referenced media
while read url; do
  wget -x -nH "https://example.com/$url" -P ./media/
done < media-inventory.txt
```

## URL Structure Preservation

Maintaining existing URLs prevents broken links from search engines and external sites.

**Common WordPress permalink structures:**
| WordPress Setting | URL Pattern | Example |
|-------------------|-------------|---------|
| Day and name | `/:year/:month/:day/:title/` | `/2024/01/15/my-post/` |
| Month and name | `/:year/:month/:title/` | `/2024/01/my-post/` |
| Post name | `/:title/` | `/my-post/` |
| Custom | varies | `/blog/:title/` |

**Best practices:**
- Check the WordPress permalink setting before migration (Settings → Permalinks)
- Configure your new platform to match the exact URL structure
- Create 301 redirects for any URLs that must change
- Test with a crawl tool (Screaming Frog, wget) to verify no 404s
- Don't forget to redirect `/feed/` to your new RSS feed URL
- Redirect `/wp-content/uploads/` paths if you reorganize media

## Migration Checklist

### Pre-Migration

- [ ] Audit the site: count posts, pages, custom post types, media files
- [ ] Identify all active plugins and what content they manage
- [ ] Document the current URL/permalink structure
- [ ] Export XML backup from WordPress admin
- [ ] Mirror the site with HTTrack or wget
- [ ] Export SEO metadata (titles, descriptions, canonical URLs)
- [ ] Export form submissions as CSV
- [ ] Download all media files locally
- [ ] Note any password-protected or members-only content

### During Migration

- [ ] Map WordPress content types to target platform equivalents
- [ ] Extract and transform content (HTML cleanup, frontmatter generation)
- [ ] Migrate media files with correct paths
- [ ] Set up URL redirects for any changed paths
- [ ] Preserve SEO metadata on each page
- [ ] Handle page builder content (render from mirror, not from shortcodes)
- [ ] Test internal links

### Post-Migration

- [ ] Crawl the new site for 404 errors
- [ ] Verify all redirects work correctly
- [ ] Check that images load on every page
- [ ] Validate RSS feeds
- [ ] Submit updated sitemap to search engines
- [ ] Monitor search console for crawl errors over 2-4 weeks
- [ ] Keep the old WordPress site accessible (read-only) for at least 30 days
