# Continuous Integration with GitHub Actions and Dependabot

Every time you push code, there is a risk of breaking something that worked before. **Continuous Integration (CI)** automates the process of checking your code on every push: running tests, linting, deploying docs. This page explains how the project uses GitHub Actions for CI and Dependabot to keep dependencies up to date.

## What is GitHub Actions?

GitHub Actions is GitHub's built-in automation platform. You define **workflows** in YAML files inside `.github/workflows/`. Each workflow specifies:

- **When** to run (on push, pull request, a schedule, etc.)
- **What** to run (a sequence of steps on a fresh virtual machine)

Every time the trigger condition is met, GitHub spins up a runner, executes the steps, and reports pass or fail directly on the commit or pull request.

## Workflow Structure

A minimal workflow looks like this:

```yaml
name: Run tests

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-python@v6
        with:
          python-version: "3.12"

      - run: pip install uv
      - run: uv sync --all-extras --dev
      - run: uv run pytest tests/
```

The key sections are:

- `on` — the trigger: here, any push to `main` or any pull request
- `jobs` — one or more parallel jobs, each running on its own machine
- `steps` — sequential commands within a job; `uses` runs a reusable action, `run` executes a shell command

## Workflows in This Project

The project has five workflows in `.github/workflows/`:

### tests.yaml — Unit Tests

Runs the test suite on every push to `main` and on every pull request. It tests across a **matrix** of operating systems and Python versions to catch platform-specific issues early:

```yaml
strategy:
  matrix:
    operating-system: ["ubuntu-latest", "windows-latest", "macos-latest"]
    python-version: ["3.11", "3.12"]
```

This means a single push triggers six parallel jobs (3 OS × 2 Python versions). Each job installs dependencies with `uv sync` and runs:

```bash
uv run coverage run -m pytest tests/
uv run coverage report -m
```

### linting.yaml — Code Linting

Runs `ruff check` on every push to `main` and on pull requests. If any file violates the code style rules, the workflow fails and the PR is blocked until it is fixed.

### data-tests.yaml — Data Validation

Only triggers when files inside `data/` change. This avoids running data validation on every code commit when the data has not changed:

```yaml
on:
  push:
    paths:
      - "data/**"
```

### deploy_docs.yaml — Documentation Deployment

Deploys the MkDocs site to GitHub Pages on every push to `main`:

```bash
uv run mkdocs gh-deploy --force -f docs/mkdocs.yaml
```

The generated site is pushed to the `gh-pages` branch automatically. No manual deployment needed.

!!! warning
    This workflow will fail silently until GitHub Pages is enabled in the repository settings. Go to **Settings → Pages**, set the source to **Deploy from a branch**, and select the `gh-pages` branch. The branch is created on the first push, but the site will not be served until you enable it here.

### pre-commit-update.yaml — Pre-commit Hook Updates

Runs on a daily schedule (`cron: '0 0 * * *'`) and automatically opens a pull request when pre-commit hook versions are outdated:

```yaml
on:
  schedule:
    - cron: '0 0 * * *'
```

This exists because Dependabot does not support updating pre-commit hooks, so this workflow fills that gap.

!!! warning
    By default, GitHub Actions is not allowed to create pull requests. Without extra configuration this workflow will fail at the `create-pull-request` step. To fix it, go to **Settings → Actions → General** and under **Workflow permissions** enable **Allow GitHub Actions to create and approve pull requests**.

## Dependabot

**Dependabot** is a GitHub service that automatically opens pull requests when your dependencies have new versions available. It is configured in `.github/dependabot.yaml`:

```yaml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"

  - package-ecosystem: "uv"
    directory: "/"
    schedule:
      interval: "weekly"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

Every week, Dependabot checks three things:

- **pip / uv** — new versions of packages in `pyproject.toml`
- **github-actions** — new versions of actions used in workflows (e.g. `actions/checkout@v6`)

When an update is found, Dependabot opens a PR with the version bump. The CI workflows then run automatically on that PR, so you can see whether the update breaks anything before merging.

!!! note
    Dependabot does not support pre-commit hooks. That is why the project has a dedicated `pre-commit-update.yaml` workflow that handles those updates separately on a daily schedule.

## Exercise

!!! example "Exercise"
    1. Push a commit to your repository and navigate to the **Actions** tab on GitHub. Watch the workflows run in real time.

    2. Intentionally introduce a linting error (e.g. an unused import) and open a pull request. Observe the `linting.yaml` workflow fail and block the PR.

    3. Fix the error, push again, and confirm all checks pass.

    4. Look at the matrix in `tests.yaml` and add Python `3.13` to the `python-version` list. Push and observe the additional jobs that appear.

    5. Check the **Security** tab on GitHub to see if Dependabot has opened any dependency update PRs. Review one and merge it if the tests pass.
