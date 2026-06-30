# GitHub Pages Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and publish Morse Walker to GitHub Pages on every push to `main` (and on manual dispatch) using the existing webpack build.

**Architecture:** A single GitHub Actions workflow runs `npm ci && npm run build` to produce `dist/`, uploads it as a Pages artifact, and deploys it via the official Pages actions. No Jekyll, no source changes, no committed build output.

**Tech Stack:** GitHub Actions, the project's existing Node/webpack build.

## Global Constraints

- Single new file only: `.github/workflows/deploy-pages.yml`. No source/code changes (the build is already subpath-ready: relative asset paths, `site.webmanifest` `start_url` is `/morsewalker/`).
- Triggers exactly: `push` to `main`, and `workflow_dispatch`.
- Workflow permissions exactly: `contents: read`, `pages: write`, `id-token: write`.
- Concurrency: group `pages`, `cancel-in-progress: false`.
- Build with `npm ci` then `npm run build`; upload artifact `path: dist`.
- Use official actions at these majors: `actions/checkout@v4`, `actions/setup-node@v4` (Node 20, npm cache), `actions/configure-pages@v5`, `actions/upload-pages-artifact@v3`, `actions/deploy-pages@v4`.
- No `.nojekyll` file (the Actions artifact is served as-is).
- Commits use `git commit --no-gpg-sign` (repo requires signing but no key is configured here).
- The husky pre-commit hook runs Prettier on `src/**`; it does not touch `.github/`, so no reformatting is expected for this file.

## File Structure

- `.github/workflows/deploy-pages.yml` — the only file. Defines the build and deploy jobs.

---

### Task 1: Add the GitHub Pages deploy workflow

**Files:**
- Create: `.github/workflows/deploy-pages.yml`

**Interfaces:**
- Consumes: the existing `npm run build` script (webpack prod build + jsdoc) which writes the site to `dist/`.
- Produces: a "Deploy to GitHub Pages" workflow that publishes `dist/` to the `github-pages` environment.

- [ ] **Step 1: Create the workflow file**

Create `.github/workflows/deploy-pages.yml` with exactly this content:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Configure Pages
        uses: actions/configure-pages@v5

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: dist

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

- [ ] **Step 2: Validate the YAML parses**

Run: `node -e "const fs=require('fs'); const s=fs.readFileSync('.github/workflows/deploy-pages.yml','utf8'); if(!/name:\s*Deploy to GitHub Pages/.test(s)) throw new Error('name missing'); if(!/workflow_dispatch:/.test(s)) throw new Error('dispatch missing'); if(!/path:\s*dist/.test(s)) throw new Error('artifact path missing'); console.log('workflow file OK');"`
Expected: prints `workflow file OK` and exits 0.

(If a YAML linter is available, e.g. `npx --yes yaml-lint .github/workflows/deploy-pages.yml`, it should also report the file as valid. This is optional — the node check above is sufficient.)

- [ ] **Step 3: Confirm the build still succeeds locally**

Run: `npm run build`
Expected: `webpack ... compiled` (only the 3 pre-existing asset-size warnings) and the jsdoc step finishes; `dist/` exists. This confirms the exact command the workflow runs is green.

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/deploy-pages.yml
git commit --no-gpg-sign -m "Add GitHub Pages deploy workflow"
```

---

## Post-merge (manual, repo owner — not part of the commit)

After this is merged to `main`:

1. In the fork's **Settings → Pages**, set **Source = "GitHub Actions"** (one-time).
2. Confirm the "Deploy to GitHub Pages" run goes green on the Actions tab (build + deploy jobs).
3. Visit `https://joshuaskorich.github.io/morsewalker/` — the app loads; dark mode, the modes (incl. Character Recognition), and audio work.
4. Optionally trigger a manual `workflow_dispatch` run and confirm it also succeeds.

## Self-Review

- **Spec coverage:** Workflow file → Task 1. Triggers (push main + dispatch), permissions, concurrency, build (`npm ci` + `npm run build`), artifact `path: dist`, deploy job, action versions → all in Task 1 Step 1 verbatim. No Jekyll / no `.nojekyll` → satisfied by the artifact-based deploy (no file added). No source changes → only `.github/workflows/deploy-pages.yml` is created. One-time Pages source toggle and post-deploy verification → captured in the post-merge section (correctly out of the commit). Out-of-scope items (custom domain, upstream, PR previews, stripping `dist/docs`) → not implemented.
- **Placeholder scan:** No TBD/TODO; the full workflow YAML and exact verification commands are present.
- **Type consistency:** Action versions and the artifact path (`dist`) match the spec's Global Constraints exactly; the `deploy` job's `${{ steps.deployment.outputs.page_url }}` references the `id: deployment` step defined in the same job.
