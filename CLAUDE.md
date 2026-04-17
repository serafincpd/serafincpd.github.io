# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal blog/portfolio site at `www.serafincpd.com.ar`, built with Jekyll and hosted on GitHub Pages. Based on the "personal-jekyll-theme" (Grayscale). Site language is Spanish.

## Development

The canonical local development method is Docker (no local Ruby/Jekyll needed):

```bash
docker-compose up
# Serves at http://localhost:4000 with live reload
```

The docker-compose setup merges `_config.yml` with `_config.dev.yml` (which sets `baseurl: ""`). Jekyll 3.8.6 is used.

If Jekyll is installed locally:
```bash
jekyll serve --config _config.yml,_config.dev.yml
```

## Architecture

### Site Structure

Single-page homepage with scrollable sections (About, Blog, Contacto) served from `index.html` using the `index` layout. Sections are defined in `_config.yml` under `pages_list`.

Posts live in `_posts/` using standard Jekyll date-prefixed filenames. Blog is paginated (5 posts/page) at `/blog/page:num/`.

### Layouts and Includes

- `_layouts/` — 7 layouts: `index`, `post`, `blog`, `about`, `category`, `tag`, `error`
- `_includes/` — 22 reusable components; most JavaScript initialization is in `_includes/js.html`

The `head.html` include loads CDN resources (jQuery 1.11.3, Bootstrap 3.3.5, Font Awesome 4.4.0, Google Fonts). No Node.js or npm involved — all JS dependencies are either CDN-loaded or committed as minified files in `js/`.

### Styling

SCSS with Jekyll's built-in SASS compilation (compressed output):
- `css/grayscale.scss` — main theme stylesheet; imports `_sass/_variables.scss` and `_sass/_mixins.scss`
- `css/timeline.scss` — timeline component
- `css/rrssb.css` — social sharing buttons (static CSS, not SCSS)

Theme palette: black (`#000`) background, cyan (`#00cdff`) accents, white text.

### JavaScript

All JS initialization is in `_includes/js.html`. Key libraries committed locally:
- `js/typed.min.js` — animated dynamic text on homepage header
- `js/hammer.min.js` — swipe gesture navigation on posts/blog
- `js/rrssb.min.js` — responsive social sharing buttons

### Configuration-driven Features

Most features are toggled in `_config.yml` via boolean flags (e.g., `dynamic-typing`, `enable-gesture-navigation`, `syntax-highlight`, social sharing toggles). Check `_config.yml` before adding feature-related code — many things are already wired up but disabled.

### SEO

SEO meta tags (OpenGraph, Twitter Cards) are in `_includes/seo-meta.html`, included from `_includes/head.html`. Posts support `image:` and `description:` front matter for OG tags.

### Plugins

- `jekyll-paginate` — blog pagination
- `jemoji` — emoji support in posts

### Content Organization

- Categories: defined as pages in `categories/` directory, using `category` layout
- Tags: defined as pages in `tags/` directory, using `tag` layout
- Post front matter supports: `title`, `description`, `image`, `categories`, `tags`
