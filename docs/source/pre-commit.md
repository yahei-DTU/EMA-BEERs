# Git Hooks with pre-commit

Every project accumulates small, recurring issues: trailing whitespace, files accidentally committed with debug changes, code that was never formatted. Catching these manually is tedious and easy to forget. **pre-commit** is a tool that runs a set of checks automatically every time you make a git commit, before the commit is actually recorded.

## What is pre-commit?

pre-commit is a framework for managing **git hooks** — scripts that git runs at certain points in the workflow. The most useful hook is `pre-commit`, which runs before a commit is recorded. If any hook fails, the commit is blocked until the issue is fixed.

Without pre-commit, a common workflow looks like this:

1. Write code
2. `git commit`
3. CI catches a linting error two minutes later
4. Fix it, commit again

With pre-commit, the linting error is caught locally before the commit even happens — no wasted CI run, no extra commit.

## Installing pre-commit

pre-commit is already in the project's dev dependencies. After `uv sync` it is available immediately. You then need to install the git hooks once per repository clone:

```bash
uv run pre-commit install
```

This registers the hooks with git. From this point on, the hooks run automatically on every `git commit`.

## The Configuration File

Hooks are configured in `.pre-commit-config.yaml` at the root of the project. The project currently has:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
```

Each entry under `hooks` is a check that runs on every commit:

| Hook | What it does |
|------|-------------|
| `trailing-whitespace` | Removes trailing whitespace from all lines |
| `end-of-file-fixer` | Ensures every file ends with a newline |
| `check-yaml` | Validates that YAML files are well-formed |
| `check-added-large-files` | Blocks commits that include files over 500 KB |

## Adding More Hooks

You can extend the config with hooks from other repositories. A common addition for this project is `ruff` for linting and formatting:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.11.2
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

After editing the config, run the hooks once on all files to apply any fixes immediately:

```bash
uv run pre-commit run --all-files
```

## What Happens on a Commit

When you run `git commit`, pre-commit intercepts it and runs each hook on the staged files:

```
trim trailing whitespace.................................................Passed
fix end of files.........................................................Passed
check yaml...............................................................Passed
check for added large files..............................................Passed
```

If a hook **fails**, the commit is blocked:

```
trim trailing whitespace.................................................Failed
- hook id: trailing-whitespace
- exit code: 1
- files were modified by this hook

src/ema_beers/train.py
```

Many hooks (like `trailing-whitespace` and `ruff-format`) automatically fix the problem. After a failure you just need to stage the fixed files and commit again:

```bash
git add .
git commit -m "your message"
```

## Running Hooks Manually

You can run all hooks on all files at any time without making a commit:

```bash
uv run pre-commit run --all-files
```

Or run a specific hook:

```bash
uv run pre-commit run trailing-whitespace --all-files
uv run pre-commit run ruff --all-files
```

This is useful after adding a new hook to apply it to the entire codebase at once.

## Keeping Hooks Up to Date

Hook versions are pinned in `.pre-commit-config.yaml` (e.g. `rev: v6.0.0`). To update all hooks to their latest versions:

```bash
uv run pre-commit autoupdate
```

This updates the `rev` fields in the config file. The project's `pre-commit-update.yaml` GitHub Actions workflow does this automatically on a daily schedule and opens a pull request with the changes.

!!! note
    `uv run pre-commit install` must be run once after every fresh clone of the repository. The hooks are stored in `.git/hooks/`, which is not committed to Git, so each new clone starts without them.

## Exercise

!!! example "Exercise"
    1. Install the pre-commit hooks in your repository:

        ```bash
        uv run pre-commit install
        ```

    2. Run all hooks on the existing codebase to check the current state:

        ```bash
        uv run pre-commit run --all-files
        ```

    3. Add a trailing space to any line in a Python file, then try to commit it. Observe how pre-commit blocks the commit and fixes the file automatically.

    4. Add the `ruff-pre-commit` hooks to `.pre-commit-config.yaml` and run them on all files:

        ```bash
        uv run pre-commit run --all-files
        ```

    5. Run `uv run pre-commit autoupdate` and check whether any hook versions were bumped in `.pre-commit-config.yaml`.
