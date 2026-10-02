# ZAANIX website

The marketing site and documentation for [ZAANIX](https://github.com/contact-ajmal/Zaanix), a self-hosted data workspace on DuckDB with an agent that works in it.

**Live:** https://contact-ajmal.github.io/Zaanix-web/

## Build

No dependencies; Node 20 or later.

```sh
node build.mjs        # → dist/, and fails if any internal link or image is broken
npx serve dist        # or any static server, to look at it
```

Every push to `main` builds the site and publishes it to GitHub Pages (`.github/workflows/pages.yml`).

## What is where

| Path | What |
|---|---|
| `pages/*.html` | Marketing pages: HTML fragments wrapped in the shared layout. `{{root}}`, `{{version}}`, `{{repo}}`, `{{hub}}`, `{{image}}` and `{{ghcr}}` are filled in at build time. |
| `docs/*.md` | Documentation, in Markdown with a small front-matter block (`title`, `order`, `group`, `description`). |
| `assets/` | Stylesheet, script and favicon. |
| `screenshots/` | Product screenshots, published as `assets/img/`. They are taken from a running ZAANIX with `scripts/site-screenshots.mjs` in the ZAANIX repository; copy the new files here. |
| `build.mjs` | The builder. `SITE` at the top holds the site's address and the project links. |
| `package.json` | `zaanixVersion`: the ZAANIX release the site describes. Change it with each release. |
