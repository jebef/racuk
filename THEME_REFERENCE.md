# Theme Reference

## Liquid Basics

In plain HTML, you write markup and the browser renders it exactly. Liquid adds a layer on top: Shopify's server reads `.liquid` files, evaluates all the `{{ }}` and `{% %}` tags, and sends the resulting plain HTML to the browser. The browser never sees Liquid.

Two constructs:

- `{{ }}` — **Output.** Prints a value. `{{ shop.name }}` becomes `Basil Racuk` in the final HTML.
- `{% %}` — **Logic.** Conditionals, loops, variable assignment. `{% if %}`, `{% for %}`, `{% assign %}`. These produce no output — they control what HTML gets generated.

**Filters** are pipe-chained transformations: `{{ 'now' | date: '%Y' }}` takes the string `'now'`, passes it through the `date` filter, outputs `2026`. Always `value | filter_name: arg`.

### Shopify Objects

Pre-populated variables Shopify gives you. Never defined manually — they exist based on context:

| Object | Available | Description |
|---|---|---|
| `shop` | Everywhere | Store name, description, URL |
| `product` | Product pages only | Current product |
| `collection` | Collection pages only | Current collection |
| `section` | Inside sections | Current section's settings and blocks |
| `block` | Inside block loops | Current block's settings and type |
| `routes` | Everywhere | URL paths (`routes.root_url`, `routes.cart_url`) |
| `request` | Everywhere | Info about the current page request |
| `settings` | Everywhere | Global theme settings from `config/settings_schema.json` |

---

## File Structure

### Entry Point: `layout/theme.liquid`

The `index.html` equivalent. Every page renders through this file.

```
<head>
  css-variables snippet  →  generates CSS custom properties
  critical.css           →  global stylesheet
  meta-tags snippet      →  SEO/social meta tags
  content_for_header     →  Shopify injects its own scripts (analytics, etc.)
</head>
<body>
  header-group           →  section group (renders the header section)
  MainContent            →  where page-specific content goes
  footer-group           →  section group (renders the footer section)
</body>
```

`{{ content_for_layout }}` is the slot where Shopify injects page content. Visiting `/` loads the index template, `/products/bag` loads the product template, etc.

### Templates: `templates/*.json`

Decide **which sections** appear on a given page type. JSON files that list sections and their settings — no HTML.

`templates/index.json` says: "The homepage has an `image-carousel` section followed by a `navbar` section." Shopify reads this JSON, finds the section files, renders them, and injects the result into `{{ content_for_layout }}`.

### Sections: `sections/*.liquid`

Self-contained components with HTML, CSS, and a `{% schema %}` that defines admin settings.

**Important:** Shopify wraps each section's HTML in `<div class="shopify-section" id="shopify-section-{id}">` automatically. This is why the `.shopify-section` grid in `critical.css` affects everything.

The `{% schema %}` at the bottom is JSON that tells the Shopify admin what controls to show. Values are saved and accessible as `section.settings.{id}`.

| Section | Used by | Purpose |
|---|---|---|
| `header.liquid` | `header-group.json` (every page) | Site logo/name |
| `footer.liquid` | `footer-group.json` (every page) | Social links, nav |
| `image-carousel.liquid` | `index.json` (homepage) | Product slideshow |
| `navbar.liquid` | `index.json` (homepage) | COLLECTION/NEWS/ABOUT nav |
| `product.liquid` | `product.json` | Product detail page |
| `collection.liquid` | `collection.json` | Product grid for a collection |
| `collections.liquid` | `collections.json` | List of all collections |
| `cart.liquid` | `cart.json` | Shopping cart |
| `article.liquid` | `article.json` | Blog post |
| `blog.liquid` | `blog.json` | Blog listing |
| `page.liquid` | `page.json` | Generic CMS page |
| `search.liquid` | `search.json` | Search results |
| `404.liquid` | `404.json` | Not found page |
| `password.liquid` | `password.json` | Store password page |
| `custom-section.liquid` | Any page via admin | Generic content section |

### Section Groups: `sections/header-group.json` and `sections/footer-group.json`

Define sections that appear on **every page** (outside `{{ content_for_layout }}`). Called in the layout with `{% sections 'header-group' %}`. JSON that references sections and their order.

### Blocks: `blocks/*.liquid`

Nested components inside sections or other blocks.

- `blocks/group.liquid` — Layout container (flex row or column). Uses `{% content_for 'blocks' %}` to render children (nestable).
- `blocks/text.liquid` — Text element with configurable style and alignment.

Blocks have schemas like sections but can't exist alone — always children of a section.

**Theme blocks** (in `blocks/` directory) are reusable across any section that opts in with `"blocks": [{ "type": "@theme" }]`. **Section blocks** are defined inline in a section's schema (like the carousel's `"slide"` type) and are specific to that section.

### Snippets: `snippets/*.liquid`

Reusable partials called with `{% render %}`. No schema, no admin UI. Receive data through explicit parameters. Isolated scope — cannot access caller's variables.

| Snippet | Called by | Purpose |
|---|---|---|
| `css-variables.liquid` | `theme.liquid` | Generates `:root` CSS custom properties from theme settings |
| `meta-tags.liquid` | `theme.liquid` | SEO meta tags, Open Graph, Twitter cards |
| `image.liquid` | Any section | Reusable responsive image component |

### Config: `config/`

- `settings_schema.json` — Defines **global** theme settings (typography, layout, colors). Accessed everywhere via `settings.{id}`.
- `settings_data.json` — Stores **current values**. Auto-generated by the admin. Empty = all defaults.

### Locales: `locales/`

- `en.default.json` — Storefront translations (what customers see). Used with `{{ 'cart.title' | t }}`.
- `en.default.schema.json` — Admin translations (what merchants see in the theme editor). When a schema says `"label": "t:labels.menu"`, it looks up `labels.menu` here.

### Assets: `assets/`

Static files served as-is. Referenced with `{{ 'filename' | asset_url }}`.

- `critical.css` — Only stylesheet. Contains reset, body layout, `.shopify-section` grid.
- SVG icons for UI elements.

---

## Data Flow

```
settings_schema.json          → defines global settings
        ↓
settings_data.json            → stores current values
        ↓
css-variables.liquid          → reads settings, outputs CSS vars
        ↓
theme.liquid                  → loads CSS, renders header/footer groups,
        ↓                       injects page content
templates/index.json          → "use carousel + navbar sections"
        ↓
sections/image-carousel.liquid → renders HTML using section.settings
                                  and section.blocks
        ↓
Browser receives              → plain HTML + CSS, no Liquid
```

---

## CSS Architecture

### The `.shopify-section` Grid

Every section gets wrapped in `<div class="shopify-section">` by Shopify. `critical.css` applies a 3-column grid to this wrapper:

```
[margin] [content] [margin]
```

- Direct children land in column 2 (centered, constrained to `--page-width`)
- Children with class `full-width` span all 3 columns (edge to edge)

This grid affects **all** sections including header and footer.

### `{% stylesheet %}` vs `{% style %}`

- `{% stylesheet %}` — Compiled into a bundled CSS file. **Deduplicated** (included once even if rendered many times). **Static** — no Liquid variables allowed.
- `{% style %}` — Renders an inline `<style>` tag. **Allows Liquid interpolation** for dynamic values. Rendered every time.

---

## Snippet Design Patterns

### Parameter passing

```liquid
{% render 'image', image: product.featured_image, class: 'my-class' %}
```

Inside the snippet, `image` and `class` are local variables. Everything must be passed explicitly.

### Documentation

```liquid
{% doc %}
  @param {image} image - The image to render
  @param {string} [url] - Optional link wrapper
{% enddoc %}
```

`{% doc %}` is for developer reference only — no runtime effect. Square brackets `[]` indicate optional params.

### Defaults

```liquid
{% liquid
  assign speed = speed | default: 5
%}
```

Use `| default:` for optional parameters with fallback values.
