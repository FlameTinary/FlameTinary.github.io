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
```

Open the local site at `http://localhost:4000`.

## Content structure

- `source/_posts/`: blog posts
- `source/about/`, `source/privacy/`, `source/tags/`, `source/categories/`: standalone pages
- `_config.yml`: site-wide Hexo config
- `_config.fluid.yml`: Fluid theme config

## Notes for maintainers

- The active theme is `fluid`, provided by the installed `hexo-theme-fluid` package. The old checked-in `themes/next` and `themes/landscape` copies have been removed because they were not used by the current build.
- Local search is provided by Fluid and generates `public/local-search.xml`. The older `search.xml` generator config has been removed to avoid duplicate search indexes.
- Run `npm run clean` before `npm run build` when removing pages or changing generated routes, so stale files do not remain in `public/`.
- The primary deployment path is [.github/workflows/deploy.yml](/Users/sheldon/CodeRepo/FlameTinary.github.io/.github/workflows/deploy.yml:1). Pushing to the `hexo` branch triggers a build and syncs the generated `public/` output to the `master` branch.
- The local `npm run deploy` command and Hexo deploy config are kept as a fallback, but the automated workflow should be the default publishing path.
