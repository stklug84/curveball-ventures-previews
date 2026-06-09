# Contributing

Thanks for your interest in contributing to **Curveball Ventures**, the Jekyll site behind [curveball-ventures.com](https://curveball-ventures.com).

This document is the practical "how do I contribute" guide. For the full architecture, CI pipeline, security model, and operational runbook, see the [README](README.md).

## Expectations

Be respectful and constructive in issues, pull requests, and reviews. Keep changes focused — one logical change per pull request.

## Prerequisites

- **Ruby** ≥ 3.1 with **Bundler** (for running the site locally)
- **Git**

CI builds the site with `actions/jekyll-build-pages`, which provides its own Ruby and Jekyll. The `Gemfile` exists so you can build and preview locally.

## Local setup and preview

```bash
bundle install
bundle exec jekyll serve
```

Open `http://localhost:4000/` to preview your changes with live reload.

## Branching model

- Branch from `main`. Direct pushes to `main` are blocked by branch protection.
- Name branches by intent: `feat/...`, `fix/...`, `chore/...`, `docs/...`.

```bash
git checkout main
git pull
git checkout -b feat/your-change
```

## Making a change

1. Create a branch from `main` (see above).
2. Commit and push to your branch:

   ```bash
   git add .
   git commit -m "Describe your change"
   git push -u origin feat/your-change
   ```

   Pushing triggers the preview workflow, which builds the site and publishes it to `https://curveball-ventures.info/<short-sha>/` within about 30 seconds.

3. Open a pull request against `main`. A bot posts a sticky comment with the preview URL, and CI runs automatically.
4. Iterate by pushing more commits — the preview and CI re-run, and the sticky comment updates to the latest commit.
5. Once the required `build` check is green and the change is approved, **squash-merge**. The branch is deleted automatically.

Merging to `main` queues a production deploy that pauses for manual approval before going live.

## Commit and pull request guidelines

- Write clear, imperative commit messages (for example, "Add hero subtitle").
- Fill out the pull request template, including the reviewer checklist.
- **Squash merge** is the only enabled merge strategy. Each pull request becomes a single commit on `main`.

## Continuous integration checks

The `pr-validate` workflow runs on every pull request:

- **`build`** — required. The pull request cannot merge until this passes.
- **`html-validate`, `link-check`, `spell-check`, `yaml-lint`, `actions-lint`, `markdown-lint`** — advisory. They do not block merging, but please try to keep them green.

## Running linters locally (optional)

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

## Spell check

If the spell checker flags a legitimate term (a name, brand, or technical word), add it to the `words` list in `.cspell.json` rather than rewording the content.

## Adding a new linter

See [How do I add a new linter to the PR pipeline?](README.md#how-do-i-add-a-new-linter-to-the-pr-pipeline) in the README for the full procedure.

## Reporting issues

Open a GitHub issue describing the problem. Include steps to reproduce and, where relevant, the preview URL for the affected commit.

---

For architecture, branching rationale, the security model, and the operational runbook, see the [README](README.md).
