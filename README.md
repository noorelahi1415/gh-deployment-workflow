# gh-deployment-workflow

A GitHub Actions workflow that automatically deploys `index.html` to GitHub Pages whenever it changes on the `main` branch — no manual deployment steps required.

Built as part of the [roadmap.sh](https://roadmap.sh/projects/github-actions-deployment-workflow) DevOps projects track.

🔗 **Live site:** https://noorelahi1415.github.io/gh-deployment-workflow/

---

## Overview

This project demonstrates a minimal but complete CI/CD pipeline using GitHub Actions and GitHub Pages. On every push to `main` that modifies `index.html`, a workflow automatically checks out the repository, packages the site, and deploys it — with zero manual intervention.

## What It Does

- Watches the `main` branch for pushes
- Triggers **only** when `index.html` changes (not on README edits, config changes, etc.)
- Checks out the repository
- Packages the site as a Pages deployment artifact
- Deploys it live to GitHub Pages

## Project Structure

```
gh-deployment-workflow/
├── index.html                    # The deployed site
├── README.md                     # This file
└── .github/
    └── workflows/
        └── deploy.yml            # The CI/CD workflow
```

## The Workflow (`deploy.yml`)

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
    paths:
      - 'index.html'

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Pages
        uses: actions/configure-pages@v5

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: '.'

      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4
```

**How it works:**
- `on.push.paths` restricts triggering to changes in `index.html` only — a README or workflow edit alone won't kick off a deployment.
- `permissions` grants the minimum access the job needs: reading the repo, writing to Pages, and issuing a secure OIDC token for authentication.
- The four steps chain together official GitHub Actions: check out the code, prepare the Pages environment, package the site as an artifact, then publish it.

## Setup (to reproduce this yourself)

1. Create a public GitHub repository.
2. Add an `index.html` file with your content.
3. Add the workflow above at `.github/workflows/deploy.yml`.
4. In the repo, go to **Settings → Pages → Build and deployment → Source**, and set it to **GitHub Actions** (not "Deploy from a branch").
5. Push to `main` — the workflow runs automatically, and the site goes live at `https://<username>.github.io/<repo-name>/`.

## Challenges Faced & How They Were Solved

Building this looked simple on paper, but three real issues came up during setup — each a useful, practical lesson in how GitHub Actions actually behaves in production:

### 1. Push rejected: missing `workflow` scope
```
! [remote rejected] main -> main (refusing to allow a Personal Access Token 
to create or update workflow `.github/workflows/deploy.yml` without `workflow` scope)
```
**Cause:** GitHub requires a Personal Access Token to explicitly carry the `workflow` scope before it will accept pushes that create or modify files inside `.github/workflows/`. A standard `repo`-scoped token isn't enough.

**Fix:** Generated a new classic PAT with both `repo` and `workflow` scopes enabled, and re-authenticated with it.

### 2. "Invalid workflow file… error in your yaml syntax on line 18"
The YAML looked completely valid on inspection — no tabs, no BOM, correct indentation. The real cause turned out to be a **stale run**: the failing run shown in the Actions tab was still pointing at an earlier, broken commit. Because the `paths: ['index.html']` filter only triggers on changes to `index.html`, a commit that fixed *only* `deploy.yml` never triggered a new run — so the old failure kept showing as the latest result.

**Fix:** Made a small change to `index.html` itself to trigger a fresh run against the corrected workflow file, confirming the fix actually applied.

### 3. "Get Pages site failed... Not Found"
Once the workflow syntax was correct and running, `actions/configure-pages@v5` still failed:
```
Error: Get Pages site failed. Please verify that the repository has Pages 
enabled and configured to build using GitHub Actions.
```
**Cause:** GitHub Pages was not yet enabled for the repository, and the deployment source was still on its default setting ("Deploy from a branch") instead of "GitHub Actions."

**Fix:** Under **Settings → Pages → Build and deployment**, changed the source to **GitHub Actions**, then re-ran the workflow — after which it deployed successfully.

**Key takeaway:** most CI/CD failures in a pipeline like this aren't code bugs — they're permissions, configuration, or trigger-condition issues that only surface once you actually run the pipeline end to end. Debugging them meant reading the exact error text carefully rather than guessing, and checking the Actions run history to understand *which* commit was actually being evaluated.

## Skills Demonstrated

- Writing and debugging GitHub Actions YAML workflows
- Configuring path-based triggers to control when a pipeline runs
- Understanding GitHub Actions permissions (`contents`, `pages`, `id-token`) and least-privilege setup
- Diagnosing PAT scope issues, stale workflow runs, and GitHub Pages configuration errors
- End-to-end CI/CD: push → build → deploy, entirely automated

## Author

Noor Elahi
