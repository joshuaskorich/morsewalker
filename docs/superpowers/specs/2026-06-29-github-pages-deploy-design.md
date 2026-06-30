# GitHub Pages Deployment — Design Spec

**Date:** 2026-06-29
**Branch:** `feature/github-pages-deploy`

## Goal

Automatically build and publish Morse Walker to GitHub Pages on the
`joshuaskorich/morsewalker` fork, using the existing webpack build — no Jekyll.

## Why not Jekyll

Morse Walker is a webpack-bundled static site (`npm run build` → `dist/`), not a
Markdown/Liquid site. Jekyll would fight the existing pipeline and add nothing.
The correct fit is GitHub's "GitHub Actions" Pages source: a workflow runs the
webpack build and deploys the `dist/` output via the official Pages actions. The
artifact is served as-is (no Jekyll step), so no `.nojekyll` file is needed.

## Already Pages-ready (no source changes)

The build output already works on a project subpath
(`https://joshuaskorich.github.io/morsewalker/`):

- All asset references in `src/index.html` are relative (`js/app.js`,
  `img/...`, `favicon.ico`, `site.webmanifest`) — verified no absolute `/...`
  asset paths.
- `src/site.webmanifest` already sets `"start_url": "/morsewalker/"`.
- `jsdoc` writes API docs to `./dist/docs` (per `jsdoc.json`), so the deployed
  artifact includes them at `/morsewalker/docs/` — harmless, left as-is.

Therefore this change is purely additive: one workflow file, no code edits.

## The workflow

New file: `.github/workflows/deploy-pages.yml`.

**Triggers:**
- `push` to `main`
- `workflow_dispatch` (manual "Run workflow" button)

**Top-level permissions** (required by the Pages deploy actions):
- `contents: read`
- `pages: write`
- `id-token: write`

**Concurrency:** group `pages`, `cancel-in-progress: false` (queue deploys
rather than cancel an in-flight publish).

**`build` job** (`runs-on: ubuntu-latest`):
1. `actions/checkout@v4`
2. `actions/setup-node@v4` with `node-version: 20` and `cache: npm`
3. `npm ci`
4. `npm run build` (existing webpack production build + jsdoc → `dist/`)
5. `actions/configure-pages@v5`
6. `actions/upload-pages-artifact@v3` with `path: dist`

**`deploy` job** (`needs: build`, `runs-on: ubuntu-latest`):
- `environment: github-pages` (with `url` from the deploy step output)
- `actions/deploy-pages@v4`

## One-time manual setup (repo owner)

In the fork's **Settings → Pages**, set **Source = "GitHub Actions"**. The
workflow's `configure-pages` step will attempt to enable Pages, but the repo
toggle is the authoritative switch and may be required once. GitHub Actions must
also be enabled for the repo.

## Result

`https://joshuaskorich.github.io/morsewalker/` serves the latest `main` build,
redeployed on every push to `main` and on manual dispatch.

## Testing

No automated test for the deploy itself. Verification after this lands on
`main`:

1. The "Deploy to GitHub Pages" Actions run completes green (build + deploy
   jobs).
2. The Pages URL loads the app, dark mode and the modes (including Character
   Recognition) work, and audio plays.
3. A manual `workflow_dispatch` run also succeeds.

The `npm run build` step is already exercised locally and in this repo, so build
failure in CI is unlikely; the unknowns are Pages enablement and the subpath,
both covered by the manual checks above.

## Out of scope

- Custom domain / CNAME.
- Deploying to the upstream `sc0tfree/morsewalker` (this targets the fork).
- Preview deploys for pull requests.
- Changing the build to strip `dist/docs` from the published artifact.
