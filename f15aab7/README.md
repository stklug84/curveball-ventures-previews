# Curveball Ventures

Source repository for the production website at **[curveball-ventures.com](https://curveball-ventures.com)** and the per-commit preview site at **[curveball-ventures.info](https://curveball-ventures.info)**.

This README documents the entire deployment architecture, branching model, CI pipeline, and operational runbook. It is intended both as onboarding material for new contributors and as a reference for the repository author.

---

## Table of contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Repositories](#repositories)
- [Domains and DNS](#domains-and-dns)
- [Local repository layout](#local-repository-layout)
- [Workflows](#workflows)
  - [Production deploy (`jekyll-gh-pages.yml`)](#production-deploy-jekyll-gh-pagesyml)
  - [Preview deploy (`preview-deploy.yml`)](#preview-deploy-preview-deployyml)
  - [PR validation (`pr-validate.yml`)](#pr-validation-pr-validateyml)
  - [PR preview comment (`pr-comment-preview.yml`)](#pr-preview-comment-pr-comment-previewyml)
- [Branching and PR process](#branching-and-pr-process)
- [Secrets and environments](#secrets-and-environments)
- [Linters and code checks](#linters-and-code-checks)
- [Local development](#local-development)
- [Operational runbook](#operational-runbook)
- [Security model](#security-model)
- [FAQ and design rationale](#faq-and-design-rationale)

---

## Overview

The site is a [Jekyll](https://jekyllrb.com/) static site hosted on GitHub Pages. Two domains are served from two separate repositories:

| Domain | Repository | Purpose | Trigger |
|---|---|---|---|
| `curveball-ventures.com` | `stklug84/curveball-ventures` (this repo) | Production website | Merge to `main` |
| `curveball-ventures.info` | `stklug84/curveball-ventures-previews` | Per-commit previews under `/<short-sha>/` | Every push, any branch |

The split exists because **GitHub Pages permits only one custom domain (`CNAME`) per repository**. Hosting both domains from one repo is not possible without an external CDN, so the previews repo is a dedicated, write-only target for build output.

The previews site also serves a plain top-level `index.html` that **HTTP-200 redirects** any visitor of `curveball-ventures.info/` to `curveball-ventures.com/`. Individual previews remain reachable only by their direct URL `curveball-ventures.info/<short-sha>/` — there is no index page listing them.

---

## Architecture

```
                  ┌────────────────────────────────────────────┐
                  │ stklug84/curveball-ventures (this repo)    │
                  │                                            │
                  │  _config.yml, index.html, assets/, CNAME   │
                  │  .github/workflows/                        │
                  │    ├── jekyll-gh-pages.yml                 │
                  │    ├── preview-deploy.yml                  │
                  │    ├── pr-validate.yml                     │
                  │    └── pr-comment-preview.yml              │
                  └────────────────┬───────────────────────────┘
                                   │
            ┌──────────────────────┼─────────────────────────┐
            │                      │                         │
   push to main             push to ANY branch       pull_request to main
            │                      │                         │
            ▼                      ▼                         ▼
  ┌─────────────────┐   ┌────────────────────────┐  ┌──────────────────────┐
  │ Jekyll build    │   │ Jekyll build           │  │ pr-validate.yml       │
  │ ↓               │   │ (baseurl=/<sha>,       │  │  ├── build (required) │
  │ deploy-pages    │   │  url=…info)            │  │  ├── html-validate    │
  │ ↓               │   │ ↓                      │  │  ├── link-check       │
  │ github-pages    │   │ commit + push to       │  │  ├── spell-check      │
  │ environment     │   │ previews repo          │  │  ├── yaml-lint        │
  │ (approval gate) │   │ ↓                      │  │  ├── actions-lint     │
  │ ↓               │   │ prune to last 20       │  │  └── markdown-lint    │
  │ curveball-      │   │ ↓                      │  │                       │
  │ ventures.com    │   │ curveball-ventures.info│  │ pr-comment-preview.yml│
  └─────────────────┘   │  /<short-sha>/         │  │  └── sticky comment   │
                        └────────────────────────┘  └──────────────────────┘
                                   │
                                   ▼
                  ┌────────────────────────────────────────────┐
                  │ stklug84/curveball-ventures-previews       │
                  │                                            │
                  │  CNAME → curveball-ventures.info           │
                  │  .nojekyll                                 │
                  │  index.html (redirect to .com)             │
                  │  <short-sha-1>/ … <short-sha-N>/           │
                  └────────────────────────────────────────────┘
```

---

## Repositories

### Source repository (this one)
`stklug84/curveball-ventures`

- Holds the Jekyll source: `_config.yml`, `index.html`, `assets/`.
- Holds all GitHub Actions workflows.
- Holds linter configuration files.
- `CNAME` = `curveball-ventures.com` — production custom domain.
- Pages source: GitHub Actions (not branch-based).
- `preview/` is **git-ignored** locally; it is a local checkout of the previews repo for convenience, never committed here.

### Previews repository
`stklug84/curveball-ventures-previews`

- Pages source: `main` branch, root.
- `CNAME` = `curveball-ventures.info`.
- `.nojekyll` present at root — tells GitHub Pages to serve files verbatim without running Jekyll on them (the content is already-built Jekyll output).
- `index.html` at root — a minimal HTML page that meta-refreshes to `curveball-ventures.com` (still HTTP 200; the redirect is client-side).
- Per-commit folders: `<short-sha>/` — one directory per preview build, populated by the source repo's workflow via cross-repo push.
- The workflow keeps the **20 most recently committed** preview directories; older ones are pruned.

---

## Domains and DNS

| Domain | Records |
|---|---|
| `curveball-ventures.com` | Apex A records → `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153` |
| `curveball-ventures.info` | Same four GitHub Pages apex A records |

Both domains terminate at GitHub Pages. The custom domain is asserted in each repository's `CNAME` file and in Pages settings. HTTPS is provided automatically by GitHub Pages (Let's Encrypt) once DNS is validated.

---

## Local repository layout

```
.
├── _config.yml                  # Jekyll config (site title only; baseurl/url set at build time)
├── index.html                   # Production landing page (Jekyll-templated, dark-mode CSS)
├── CNAME                        # curveball-ventures.com
├── assets/
│   └── favicon.svg              # Hand-drawn curveball SVG icon
├── preview/                     # GIT-IGNORED — local clone of the previews repo
│   ├── .nojekyll
│   ├── CNAME                    # curveball-ventures.info
│   └── index.html               # Redirect to .com
├── .github/
│   ├── CODEOWNERS               # Auto-request review from @stklug84
│   ├── dependabot.yml           # Weekly Action version updates
│   ├── pull_request_template.md # Default PR body
│   └── workflows/
│       ├── jekyll-gh-pages.yml      # Production deploy
│       ├── preview-deploy.yml       # Per-commit preview deploy
│       ├── pr-validate.yml          # PR build + lint pipeline
│       └── pr-comment-preview.yml   # PR sticky comment with preview URL
├── .cspell.json                 # Spell-check config + allowlist
├── .htmlvalidate.json           # HTML validator ruleset
├── .lycheeignore                # Link-checker URL exclusions
├── .markdownlint.jsonc          # Markdown lint rules
├── .yamllint.yml                # YAML lint rules
└── .gitignore                   # Excludes preview/, VSCodium files, etc.
```

---

## Workflows

All four workflows live in `.github/workflows/`. They are intentionally decoupled — each has a single responsibility — so failures or changes in one do not cascade.

### Production deploy (`jekyll-gh-pages.yml`)

**Triggers**
- `push` to `main` (the normal case — happens automatically after a PR merge)
- `workflow_dispatch` (manual escape hatch from the Actions tab)

**Steps**
1. **Build job**: checkout, `actions/configure-pages`, `actions/jekyll-build-pages` against repo root → `_site/`, upload artifact.
2. **Deploy job**: `actions/deploy-pages` publishes the artifact. The job declares `environment: github-pages`, which means the GitHub-managed `github-pages` environment gates the deploy.

**Why the environment matters**
The `github-pages` environment is configured (manually, in repo settings) with:
- **Required reviewer**: `@stklug84` — every production deploy pauses for explicit approval.
- **Deployment branches**: `main` only.

This means even an accidental merge into `main` cannot reach the live site without a deliberate human click. The approval appears in the Actions run and as a deployment notification.

**Concurrency**
The workflow uses `concurrency: { group: "pages", cancel-in-progress: false }`. Multiple in-flight production deploys queue rather than cancel; the most recent build wins, but neither is interrupted mid-deploy.

### Preview deploy (`preview-deploy.yml`)

**Triggers**
- `push` to any branch (`branches: ['**']`)
- `workflow_dispatch`

**Steps**
1. Checkout the source repo with `fetch-depth: 0`.
2. Compute `SHORT_SHA=${GITHUB_SHA:0:7}` and export to `$GITHUB_ENV`.
3. **Override `_config.yml`** runner-locally (never committed back):
   - Append `baseurl: /<SHORT_SHA>` so all `relative_url` references in templates resolve under the subdirectory.
   - Append `url: https://curveball-ventures.info` so absolute URLs use the preview domain.
   - If `_config.yml` did not exist, it is created with just these two keys.
4. Build with `actions/jekyll-build-pages@v1` → `_site/`.
5. Checkout the **previews repo** into `previews/` using the cross-repo PAT (`secrets.PREVIEWS_DEPLOY_TOKEN`).
6. Stage the build: `rm -rf previews/<SHORT_SHA>` then `cp -R _site/. previews/<SHORT_SHA>/`. Idempotent for re-runs of the same SHA.
7. **Prune** to the newest 20 preview directories:
   - For each immediate subdirectory (excluding `.git`, `CNAME`, `.nojekyll`, `index.html`), read its last-commit timestamp via `git log -1 --format=%ct -- <dir>`.
   - Directories not yet committed (this run's fresh one, or any other uncommitted dir) are treated as "now" so they are never pruned in their own run.
   - Sort descending by timestamp, keep the first 20, `rm -rf` the rest.
8. Commit and push: identity is `github-actions[bot]`, message is `Preview: <SHORT_SHA> from <branch>`. Step is a no-op if nothing changed.
9. Write a job summary with the preview URL, branch, and commit.

**Environment binding**
The `preview` job declares `environment: previews`. The `previews` environment (configured in repo settings) holds `PREVIEWS_DEPLOY_TOKEN` and explicitly allows **all branches**. This scoping means workflows that do not declare `environment: previews` cannot read the PAT, reducing accidental exposure.

**Concurrency**
`concurrency: { group: preview-${{ github.ref }}, cancel-in-progress: false }` — one preview build per branch at a time, no cancellation of in-flight builds. Every commit's preview is preserved.

**Baseurl details**
The `baseurl` override is essential. Without it, a preview at `/abc1234/` would generate `<link href="/assets/favicon.svg">` (resolving to the previews-repo root, returning 404). With `baseurl: /abc1234`, Jekyll's `relative_url` filter expands it to `/abc1234/assets/favicon.svg`, which is correct.

### PR validation (`pr-validate.yml`)

**Triggers**
- `pull_request` to `main` on `[opened, synchronize, reopened]`

**Concurrency**
`pr-validate-${{ pr.number }}` with `cancel-in-progress: true` — new pushes to the PR branch cancel stale lint runs.

**Jobs**

| Job | Required? | What it does |
|---|---|---|
| `build` | ✅ Yes (branch protection) | Builds the Jekyll site exactly as production would; uploads `_site/` artifact. |
| `html-validate` | Advisory | Downloads artifact; runs `html-validate` against `_site/**/*.html` using `.htmlvalidate.json`. |
| `link-check` | Advisory | Runs `lycheeverse/lychee-action@v2` against `_site/` with caching to detect broken internal/external links. Honors `.lycheeignore`. |
| `spell-check` | Advisory | Runs `streetsidesoftware/cspell-action@v6` against tracked text files using `.cspell.json`. |
| `yaml-lint` | Advisory | `yamllint -c .yamllint.yml .github/ _config.yml`. |
| `actions-lint` | Advisory | `raven-actions/actionlint@v2` against `.github/workflows/*.yml`. Catches workflow syntax errors and shellcheck issues in `run:` blocks. |
| `markdown-lint` | Advisory | `DavidAnson/markdownlint-cli2-action@v18` against `**/*.md`. |

**Required vs advisory**
Only `pr-validate / build` is wired as a required status check in branch protection on `main`. The lint jobs use `continue-on-error: true`, so their failures are visible in the PR but do not block merging. Any of them can be promoted to "required" later by editing the branch protection rule. The split lets the linter set evolve without immediately gating merges.

### PR preview comment (`pr-comment-preview.yml`)

**Triggers**
- `pull_request` to `main` on `[opened, synchronize, reopened]`

**Steps**
1. Compute the short SHA from `github.event.pull_request.head.sha`.
2. Use `marocchino/sticky-pull-request-comment@v2` with `header: preview-url` to upsert a single sticky comment on the PR containing the deterministic preview URL (`https://curveball-ventures.info/<short-sha>/`), branch name, and full commit SHA.

**Why a separate workflow**
This workflow is intentionally decoupled from `preview-deploy.yml`. The preview URL is **deterministic** — it is `<short-sha>` of the PR head — so the comment can be posted immediately without waiting for the deploy to finish. The note in the comment ("Preview may take ~30s to become available") manages user expectation. Keeping comment-and-deploy in separate workflows means a comment-step failure does not delay deploys, and a deploy failure does not silently leave the PR without a URL.

**Sticky comment behavior**
The `header: preview-url` key tells the action to find any prior comment with that header on the PR and update it in place, rather than appending a new comment every push. The PR ends with exactly one preview comment, always reflecting the latest commit.

---

## Branching and PR process

### Branching model
- `main` — always reflects what is live on `curveball-ventures.com`. Protected.
- Feature branches — `feat/...`, `fix/...`, `chore/...`, `docs/...`. Created from `main`.
- Direct pushes to `main` are not allowed by branch protection. The only way changes reach `main` is via PR + squash merge.

### Lifecycle of a change

1. **Create branch** from `main`:
   ```bash
   git checkout main
   git pull
   git checkout -b feat/landing-copy
   ```
2. **Commit and push** to the feature branch:
   ```bash
   git add .
   git commit -m "Add hero subtitle"
   git push -u origin feat/landing-copy
   ```
   On push, `preview-deploy.yml` fires, building the site and pushing it to `curveball-ventures-previews` under `/<short-sha>/`. The preview is reachable within ~30s.
3. **Open a PR** to `main`. The PR template populates the body with a reviewer checklist.
   - `pr-validate.yml` runs the build (required) and all linters (advisory).
   - `pr-comment-preview.yml` posts a sticky comment with the preview URL.
   - CODEOWNERS auto-requests review from `@stklug84`.
4. **Review**: open the preview URL, walk through the checklist. Iterate by pushing more commits to the branch — both workflows re-run, and the sticky comment updates.
5. **Merge**: once approvals and the required `build` status check are green, **squash-merge**. The branch is auto-deleted.
6. **Deploy**: merging to `main` triggers `jekyll-gh-pages.yml`. It builds and queues a deployment to the `github-pages` environment, which **pauses for approval**. The approver (you) reviews the production deploy in the Actions UI and clicks approve. The site goes live.

### Merge strategy
**Squash merge** is the only enabled strategy in repository settings. Each PR becomes one commit on `main`, named after the PR title. Consequences:
- `main` history is linear and clean.
- The commit on `main` has a **different SHA** than the last commit on the feature branch. The `.info/<sha>/` preview that was reviewed remains at the feature branch's SHA; after merge a new preview is built at the squash commit's SHA. Both work; the URL just changes.
- Reverting a PR is a single commit revert.

### Branch protection on `main`

Configured in **Settings → Branches → Branch protection rules**:

- ✅ Require a pull request before merging
  - Required approvals: **0** (raise to 1 when a second contributor joins)
  - Dismiss stale PR approvals when new commits are pushed
  - Require review from Code Owners
- ✅ Require status checks to pass before merging
  - Require branches to be up to date before merging
  - Required check: `pr-validate / build`
- ✅ Require linear history
- ✅ Do not allow bypassing the above settings
  - Include administrators (rules apply even to the repo owner)
- ❌ Allow force pushes
- ❌ Allow deletions

### Repo-level merge settings

In **Settings → General → Pull Requests**:

- ❌ Allow merge commits
- ✅ Allow squash merging — default commit title: PR title; default message: PR description
- ❌ Allow rebase merging
- ✅ Automatically delete head branches after merge

---

## Secrets and environments

### Environments

| Environment | Purpose | Branches | Reviewers | Secrets |
|---|---|---|---|---|
| `github-pages` | Production deploy to `.com` | `main` only | `@stklug84` (required) | none (uses `GITHUB_TOKEN`) |
| `previews` | Per-commit preview deploy to `.info` | All branches | none | `PREVIEWS_DEPLOY_TOKEN` |

The `github-pages` environment is auto-created by the `actions/deploy-pages` action. Configure required reviewers and branch restrictions manually in the UI.

The `previews` environment is created manually. It must allow **all branches** because the preview workflow runs on every branch push by design.

### `PREVIEWS_DEPLOY_TOKEN`

A fine-grained GitHub Personal Access Token (PAT) used by the source repo's preview workflow to push build output into `stklug84/curveball-ventures-previews`.

- **Scope**: only `stklug84/curveball-ventures-previews`.
- **Permissions**: `Contents: Read and write`.
- **Expiration**: maximum 1 year. Set a calendar reminder to rotate.
- **Storage**: environment secret on the `previews` environment, not a repository-wide secret. This scopes its availability to jobs that declare `environment: previews`.

### Why an environment secret instead of a repo secret

A repository secret is exposed to **every** workflow in the repo. An environment secret is exposed only to jobs that declare that environment. Combined with CODEOWNERS protection on `.github/workflows/**` (which requires the author's review on any workflow change), this closes the practical exfiltration paths a new contributor would otherwise have access to.

---

## Linters and code checks

Each linter has a dedicated config file at the repo root. All are wired into `pr-validate.yml` as advisory jobs.

### `.htmlvalidate.json`
Extends `html-validate:recommended`. Currently relaxed: `no-inline-style: "warn"` (the landing page uses an inline `<style>` block), `void-style: "off"`, `require-sri: "off"`. Tighten as content grows.

### `.cspell.json`
Project allowlist with terms like `Curveball`, `Jekyll`, `stklug`, `lychee`, `actionlint`. Ignores `_site/**`, `preview/**`, `.git/**`, `node_modules/**`.

### `.markdownlint.jsonc`
Permissive markdown rules. `MD013` (line length), `MD033` (inline HTML — needed for Jekyll-flavored markdown), and `MD041` (first line must be h1) are disabled.

### `.yamllint.yml`
Relaxed: `line-length` is a warning at 200 chars, `truthy` disabled, `document-start` disabled (workflow files don't start with `---`).

### `.lycheeignore`
Initially empty. Add regex patterns of URLs to skip when external links prove flaky.

### `actionlint`
No config file — runs against all workflow files with defaults. Catches shell-injection patterns and YAML schema errors in workflows.

---

## Local development

### Prerequisites
- Ruby ≥ 3.1 with Bundler (only if you want to run Jekyll locally; the CI uses `actions/jekyll-build-pages` which provides Ruby).
- Git.

### Build the site locally
There is no `Gemfile` checked in (the production build uses GitHub's preinstalled Jekyll image). To preview locally:
```bash
gem install jekyll
jekyll serve
```
Open `http://localhost:4000/`.

### Working with the previews repo
The `preview/` directory is git-ignored in this repo but is a checkout of `stklug84/curveball-ventures-previews`. If you want to modify the previews repo's `index.html`, `CNAME`, or `.nojekyll`, do so inside `preview/` and push from there:
```bash
cd preview
git pull
# edit files
git add .
git commit -m "Update redirect target"
git push
```
The preview workflow only writes to `<short-sha>/` subdirectories; it never modifies the previews repo's root files.

### Running linters locally (optional)
```bash
# YAML
pipx install yamllint
yamllint -c .yamllint.yml .github/ _config.yml

# Spelling
npm install -g cspell
cspell "**/*.{md,html,yml,yaml}"

# HTML (after a Jekyll build into _site/)
npm install --no-save html-validate
npx html-validate "_site/**/*.html"

# Workflows
brew install actionlint
actionlint
```

---

## Operational runbook

### "I want to ship a change to the live site"
1. Branch from `main`.
2. Commit, push.
3. Open a PR. Watch the sticky comment appear; open the preview URL.
4. Iterate until happy.
5. Merge (squash). Approve the production deploy when the `github-pages` environment prompt appears in the Actions tab.

### "The preview workflow is failing with 403 / authentication errors"
The PAT has likely expired or been revoked. Generate a new fine-grained PAT, update the `PREVIEWS_DEPLOY_TOKEN` secret in the `previews` environment, and re-run the failed workflow.

### "The production deploy is stuck waiting"
It is paused at the `github-pages` environment approval gate. Open the run in Actions → Review deployments → Approve.

### "I need to roll back production"
- **Preferred**: revert the offending commit on `main` via a new PR. Merging the revert PR triggers a new deploy.
- **Hot path**: re-run the most recent successful `jekyll-gh-pages.yml` workflow run via `workflow_dispatch` from the previous good commit. Branch protection still applies — the workflow runs against whatever commit `main` currently points to, so a true rollback still requires a revert PR.

### "I want to delete an old preview manually"
The pruning step keeps the newest 20 by git timestamp. To force a delete:
```bash
cd preview
git pull
rm -rf <short-sha>
git add -A
git commit -m "Remove stale preview <short-sha>"
git push
```

### "I want to force a fresh production deploy without a code change"
Actions tab → `Deploy Jekyll with GitHub Pages dependencies preinstalled` → Run workflow → branch `main`. This is the `workflow_dispatch` escape hatch.

### "The DNS / HTTPS cert is failing for one of the domains"
Check Settings → Pages in the respective repository. Re-validate the custom domain. Wait up to 24h for certificate provisioning.

---

## Security model

### Trust boundaries
- **`main` is trusted**. Branch protection ensures every commit on `main` arrived via PR with the required `pr-validate / build` check and a code-owner review.
- **Other branches are semi-trusted**. Anyone with write access can push. Their commits trigger the preview workflow but not production.
- **PRs from forks** (currently impossible because the repo is single-account, but applicable if it goes public): GitHub does not expose secrets to fork PR workflows by default. The preview workflow would not run for fork PRs unless explicitly configured otherwise.

### PAT exposure surface
Anyone who can edit `.github/workflows/preview-deploy.yml` and merge that change can read `PREVIEWS_DEPLOY_TOKEN`. Defenses in depth:
1. **CODEOWNERS** on `.github/workflows/**` and `.github/CODEOWNERS` itself — requires code-owner review on any modification.
2. **Branch protection** requires code-owner review and a passing status check.
3. **Environment secret** scoping — only jobs declaring `environment: previews` can read the PAT, so a maliciously added new workflow has to also declare the environment, which is more visible in review.
4. **Fine-grained PAT** scoped to only the previews repo — even if leaked, blast radius is bounded.

### What this design does *not* protect against
- A compromised author account.
- A malicious code-owner approving their own bad PR (if approvals are raised to 1 from a collaborator).
- The PAT being leaked outside GitHub (e.g. pasted into chat). Rotate immediately if suspected.

### Hardening upgrades to consider later
- Replace the PAT with a **GitHub App installation token**: short-lived (~1h), auto-rotating, explicit per-repo permissions.
- Enable **secret scanning** and **push protection** in the repo.
- Add **CodeQL** analysis for any future JavaScript/Ruby code.

---

## FAQ and design rationale

### Why two domains and two repos?
GitHub Pages allows only one custom domain per repository. We need both a stable production URL and per-commit preview URLs accessible simultaneously. Two repos is the simplest solution that avoids introducing a third-party host (Cloudflare Pages, Netlify, etc.).

### Why not Cloudflare Pages or Netlify for previews?
Considered and rejected. Either would give native per-commit preview URLs out of the box, but introduces a third vendor, a separate auth model, and pricing/quota considerations. The two-repo approach keeps everything inside GitHub.

### Why not a git submodule for the previews repo?
Considered and rejected. A submodule pins a specific commit, meaning the source repo would need a follow-up commit after every preview push to update the pointer. This adds noise to `main`'s history without operational benefit. The current design uses a runtime `actions/checkout` of the previews repo, which has identical access without coupling.

### Why is the previews repo's `index.html` a meta-refresh redirect and not a directory listing?
By design: we wanted preview URLs to be **unguessable** by random visitors. The root page sends visitors back to `curveball-ventures.com`. Individual previews remain accessible to anyone who has the URL (the PR sticky comment, the workflow summary, or a direct share), but are not enumerable from the root.

### Why HTTP 200 with a meta-refresh instead of a real HTTP 301?
GitHub Pages does not allow custom HTTP status codes for arbitrary paths. A real 301 would require either the `jekyll-redirect-from` plugin (overkill — would re-enable Jekyll on the previews repo, conflicting with `.nojekyll`) or an external host. Meta-refresh is functionally equivalent for users and well-understood by search engines, especially when paired with `<link rel="canonical">`.

### Why is the build job the only required CI check?
A clean build is the minimum signal we trust: it proves the Jekyll source is well-formed and produces deployable output. Lint failures are real but often noisy on a new codebase; making them all required up front would block legitimate PRs over cosmetic issues. The advisory tier gives reviewers signal without strict gates. Promote any job to required by editing branch protection — it's a one-checkbox change.

### Why is `prefers-color-scheme` dark-mode support inline in the page?
The site is currently a single page with a heading. An external stylesheet would add one HTTP request and a separate file to manage for ~15 lines of CSS. As the site grows, extract to `assets/main.css`.

### Why does the workflow only keep 20 previews?
Pragmatic balance: enough to compare branches across a sprint, few enough that the previews repo doesn't bloat. Adjust by editing `KEEP_LAST` in `.github/workflows/preview-deploy.yml`.

### What happens if I push the same commit twice (e.g. force-push, rebase)?
The preview workflow runs again. The `<short-sha>` directory in the previews repo is deleted and re-created; the commit message in the previews repo reflects the new push. No corruption, no duplicate previews.

### What if two branches contain the same commit SHA?
Both pushes target the same `<short-sha>/` folder. The first push deploys, the second push sees no content change and skips the commit (`git diff --cached --quiet && exit 0`). The preview reflects whichever ran first; both branches' PR comments point to the same valid URL.

### Why is the `.com` build's `_config.yml` minimal (just `title:`)?
The production deploy serves the site at the apex of `curveball-ventures.com`, so `baseurl` is empty and `url` would only matter for absolute URL helpers (which the current page doesn't use). The preview workflow adds those keys at build time. Keeping `_config.yml` minimal avoids confusion about whether values apply to prod, preview, or both.

### How do I add a new linter to the PR pipeline?
1. Add the config file at the repo root if needed.
2. Add a job to `.github/workflows/pr-validate.yml`, modeled after the existing ones (`continue-on-error: true` to start).
3. Open a PR. Verify it runs.
4. Optionally promote to required via branch protection once you trust its signal.

---

## Quick reference

### Key URLs
- Production: <https://curveball-ventures.com>
- Previews root (redirects): <https://curveball-ventures.info>
- Preview pattern: `https://curveball-ventures.info/<short-sha>/`

### Key files
- `.github/workflows/jekyll-gh-pages.yml` — production deploy
- `.github/workflows/preview-deploy.yml` — preview deploy
- `.github/workflows/pr-validate.yml` — PR CI
- `.github/workflows/pr-comment-preview.yml` — PR sticky comment
- `.github/CODEOWNERS` — review routing
- `_config.yml` — Jekyll site config (title only)
- `index.html` — production landing page

### Key commands
```bash
# Start a change
git checkout -b feat/your-thing main

# Push (triggers preview)
git push -u origin feat/your-thing

# Sync the local previews repo
cd preview && git pull && cd ..

# Manual production deploy (escape hatch)
# Actions tab → "Deploy Jekyll with GitHub Pages dependencies preinstalled" → Run workflow
```

### Maintenance calendar
- **PAT rotation**: ~2 weeks before `PREVIEWS_DEPLOY_TOKEN` expires (max 1 year). GitHub emails reminders.
- **Dependabot PRs**: weekly, merge after `pr-validate / build` passes.
- **Lint config review**: revisit advisory linters quarterly; promote stable ones to required.

---

_Last updated: 2026-05-22_
