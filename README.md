# FlameTinary.github.io

This repository contains the Hexo source for `https://flametinary.github.io`.

## Stack

- Hexo `8`
- Theme: `hexo-theme-fluid`
- Deployment: GitHub Actions to `master`

## Common commands

```bash
npm install
npm run clean
npm run build
npm run server
npm run deploy
```

Open the local site at `http://localhost:4000`.

## Content structure

- `source/_posts/`: blog posts
- `source/about/`, `source/privacy/`, `source/tags/`, `source/categories/`: standalone pages
- `_config.yml`: site-wide Hexo config
- `_config.fluid.yml`: Fluid theme config

## Adding custom static pages

In addition to blog posts and pages rendered by Hexo and the Fluid theme, this repository can host fully independent product landing pages, support pages, and privacy policies. The current pages are:

- Dayvu landing page: `/dayvu/`, sourced from `source/dayvu/index.html`
- Dayvu support page: `/dayvu-support/`, sourced from `source/dayvu-support/index.html`
- TShot privacy policy: `/privacy/`, sourced from `source/privacy/index.html`
- TShot support page: `/support/`, sourced from `source/support/index.html`

Maintain these pages as source files on the `hexo` branch; never edit the `master` branch directly. `master` is the generated deployment branch. Each deployment fully synchronizes a new `public/` directory, so any files written directly to `master` will be removed.

### Create a page

To add a product page named `my-product`:

1. Create the entry file at `source/my-product/index.html`.
2. Write the standalone HTML, CSS, and JavaScript in that directory. The page will be served at `/my-product/`.
3. Keep private page assets in the same directory, such as `source/my-product/assets/logo.svg`, and reference them with relative paths such as `assets/logo.svg`.
4. Add `my-product/**` to the `skip_render` list in `_config.yml`. Hexo will then copy the entire directory unchanged rather than treating `index.html` as a blog page and applying the Fluid theme.

Example configuration:

```yml
skip_render:
  - privacy/**
  - support/**
  - dayvu/**
  - dayvu-support/**
  - my-product/**
```

`skip_render` must cover the page directory and all of its children. Without it, Hexo processes the HTML and may add the blog navigation, banner, and theme styles, breaking the page's independent layout.

### Verify and publish locally

After adding or changing a custom page, run:

```bash
npm run clean
npm run build
npm run server
```

Open `http://localhost:4000/my-product/` in a browser and confirm that styles, images, and internal links work correctly. After building, you can also confirm that `public/my-product/index.html` exists and matches `source/my-product/index.html`.

After verification, commit and push `source/my-product/`, `_config.yml`, and any related assets to the `hexo` branch. GitHub Actions regenerates `public/` and synchronizes it to `master`; because the custom page is now a build output, automated deployment will preserve it.

### When not to use `skip_render`

If a page should use the blog navigation, dark mode, table of contents, and Fluid layout, create a Markdown page such as `source/about/index.md` and let Hexo render it normally. Use `index.html` with `skip_render` only for pages that need an independent HTML design or do not depend on the blog theme.

## Notes for maintainers

- The active theme is `fluid`, provided by the installed `hexo-theme-fluid` package. The old checked-in `themes/next` and `themes/landscape` copies have been removed because they were not used by the current build.
- Local search is provided by Fluid and generates `public/local-search.xml`. The older `search.xml` generator config has been removed to avoid duplicate search indexes.
- Run `npm run clean` before `npm run build` when removing pages or changing generated routes, so stale files do not remain in `public/`.
- The primary deployment path is [.github/workflows/deploy.yml](/Users/sheldon/CodeRepo/FlameTinary.github.io/.github/workflows/deploy.yml:1). Pushing to the `hexo` branch triggers a build and syncs the generated `public/` output to the `master` branch.
- The local `npm run deploy` command and Hexo deploy config are kept as a fallback, but the automated workflow should be the default publishing path.

## Publishing

- Recommended: commit and push changes to the `hexo` branch. GitHub Actions will build the site and publish the generated output to `master`.
- Fallback: run `npm run deploy` locally if you need to publish manually.
