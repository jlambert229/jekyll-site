# Foggy Clouds — Jekyll

![Jekyll](https://github.com/jlambert229/jekyll-site/actions/workflows/jekyll.yml/badge.svg)
![pre-commit](https://github.com/jlambert229/jekyll-site/actions/workflows/pre-commit.yml/badge.svg)
![GitHub last commit](https://img.shields.io/github/last-commit/jlambert229/jekyll-site)

A blog about learning cloud and DevOps technology. Converted from Hugo (Blowfish theme) to Jekyll (Beautiful Jekyll theme).

## Setup

```bash
# Install Ruby (if not installed)
# See: https://www.ruby-lang.org/en/downloads/

# Install dependencies
bundle install

# Serve locally
bundle exec jekyll serve

# Build for production
bundle exec jekyll build
```

Site will be available at `http://localhost:4000`.

## Structure

```
├── _config.yml          # Site configuration
├── _posts/              # Blog posts (YYYY-MM-DD-slug.md)
├── _includes/           # Reusable components (admonition.html)
├── assets/
│   ├── css/             # Custom CSS (admonitions)
│   └── img/             # Images and memes
├── index.html           # Homepage
├── aboutme.md           # About page
├── posts.md             # All posts listing
├── series.md            # Posts grouped by series
├── tags.html            # Posts grouped by tag
├── Gemfile              # Ruby dependencies
└── README.md
```

## Theme

Using [Beautiful Jekyll](https://beautifuljekyll.com/) v6.0.1.

## Custom Features

- **Admonition blocks** — Tip, Note, Warning, Info, Production, and Danger callout boxes via `_includes/admonition.html` and `assets/css/admonitions.css`
