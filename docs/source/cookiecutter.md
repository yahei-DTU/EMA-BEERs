# Project Templating with Cookiecutter

Starting a new project always involves the same repetitive setup: creating the right folder structure, adding a `.gitignore`, wiring up a test suite, configuring linting. **Cookiecutter** automates this by letting you generate a complete project from a template with a single command.

## What is Cookiecutter?

Cookiecutter is a command-line tool that generates project directories from templates. A template is just a regular directory structure where filenames and file contents can contain **placeholders** — variables that get filled in when you run the tool.

For example, a template might have a folder literally named `{{ cookiecutter.repo_name }}`. When you run cookiecutter, it asks you what the repo name should be, and creates a folder with that name.

## Installing Cookiecutter

Cookiecutter is a standalone tool, so you install it once and use it across all projects:

```bash
uv tool install cookiecutter
```

This makes the `cookiecutter` command available globally without polluting any project environment.

## Using a Template

The basic command is:

```bash
cookiecutter <template-url-or-path>
```

Cookiecutter will prompt you to fill in each variable defined by the template. When done, it generates the full project directory in your current location.

### Example: Using the course template

This course has a ready-made template at [https://github.com/yahei-DTU/project-template](https://github.com/yahei-DTU/project-template). To use it:

```bash
cookiecutter https://github.com/yahei-DTU/project-template
```

You will be prompted to fill in the following fields:

```
repo_name [repo_name]: my-ml-project
project_name [project_name]: My ML Project
author_name [Your name (or your organization/company/team)]: Jane Doe
author_email [Your email address]: jane@example.com
description [A short description of the project.]: Predicting bike rentals with regression
keywords [comma-separated, list, of, keywords]: ml, regression, bikes
python_version [3.12]:
Select open_source_license:
1 - No license file
2 - MIT
3 - BSD-3-Clause
Choose from 1, 2, 3 [1]: 2
```

Press Enter to accept the default shown in brackets, or type your own value. After answering all prompts, cookiecutter generates a fully structured project:

```
my-ml-project/
    .devcontainer/
    .github/
    .gitignore
    .pre-commit-config.yaml
    .python-version
    AGENTS.md
    LICENSE
    README.md
    configs/
    dockerfiles/
    docs/
    models/
    notebooks/
    reports/
    src/
    tasks.py
    tests/
    pyproject.toml
    requirements.txt
    requirements_dev.txt
```

Everything is already in place. The next step is just:

```bash
cd my-ml-project
uv sync
```

## Starting a New Project and Repository

The generated folder is not yet a git repository and isn't on GitHub. A common question is whether to create the GitHub repository first or run cookiecutter first.

**Run cookiecutter first, then create an empty repository on GitHub and push to it.** This order avoids two common problems:

- Cookiecutter always creates a *new* folder. If you clone a GitHub repo first and run cookiecutter inside it, you end up with the project nested one level too deep (`my-ml-project/my-ml-project/`).
- If the GitHub repo already has a README, `.gitignore` or license, it has its own history that conflicts with the files the template generates.

The full workflow is: **cookiecutter → `git init` + first commit → create empty GitHub repo → `git remote add` + `git push`**.

### 1. Generate the project locally

```bash
cookiecutter https://github.com/yahei-DTU/project-template
cd my-ml-project
uv sync
```

Running `uv sync` before the first commit creates `uv.lock`, so the lock file is included from the start.

### 2. Turn the folder into a git repository

```bash
git init -b main
git add .
git commit -m "Initial commit from project template"
```

`git init -b main` creates the repository with `main` as the default branch, matching GitHub's default. The template's `.gitignore` already excludes `.venv/`, `.env` and other files that should not be committed, so `git add .` is safe.

### 3. Create an empty repository on GitHub

1. Go to [github.com/new](https://github.com/new).
2. Choose the owner (your account or an organization) and use the same name as your local folder, e.g. `my-ml-project`.
3. Choose *Public* or *Private*.
4. **Leave everything under "Initialize this repository with" unticked**: no README, no `.gitignore`, no license. The template already provides all of these.
5. Click *Create repository*.

### 4. Connect the local repository to GitHub and push

GitHub shows the commands on the page of your new empty repository. They boil down to:

```bash
git remote add origin https://github.com/<your-username>/my-ml-project.git
git push -u origin main
```

- `git remote add origin <url>` tells your local repository where the remote lives. `origin` is the conventional name for the main remote.
- `git push -u origin main` uploads your commits and sets `origin/main` as the upstream of your local `main`. From then on, a plain `git push` and `git pull` are enough.

Check the connection with:

```bash
git remote -v
```

Refresh the GitHub page and your project should be there.

!!! note "Authentication"
    GitHub no longer accepts your account password for `git push` over HTTPS. The easiest fix is to install the
    [GitHub CLI](https://cli.github.com/) and run `gh auth login` once, which also configures git to use your
    GitHub credentials. Alternatively, set up an [SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
    and use the SSH URL instead: `git@github.com:<your-username>/my-ml-project.git`.

### Shortcut: steps 3 and 4 with the GitHub CLI

If you have the GitHub CLI installed and logged in, you can create the remote repository, connect it and push in a single command from inside your project folder (after step 2):

```bash
gh repo create my-ml-project --private --source=. --remote=origin --push
```

Use `--public` instead of `--private` for a public repository.

### Already created the GitHub repository first?

??? question "I created the repo on GitHub with a README (or .gitignore/license) and now `git push` is rejected"
    The remote has a commit your local repository doesn't know about. If you haven't done anything in the GitHub repo
    yet, the simplest fix is to delete it and create it again *empty*. Otherwise, merge the two histories once and keep
    the template's versions of any conflicting files:

    ```bash
    git pull origin main --allow-unrelated-histories
    # resolve any conflicts (e.g. README.md, .gitignore), then:
    git add .
    git commit
    git push -u origin main
    ```

??? question "I already cloned the GitHub repo and want to fill it with the template"
    Run cookiecutter from the folder *containing* the clone, give `repo_name` exactly the same name as the cloned folder,
    and pass `-f` (`--overwrite-if-exists`) so cookiecutter writes into the existing folder instead of failing:

    ```bash
    cookiecutter -f https://github.com/yahei-DTU/project-template
    cd my-ml-project
    git add .
    git commit -m "Add project template"
    git push
    ```

    Files with the same name, such as a README created by GitHub, are overwritten by the template's version.

## How Templates Work

A cookiecutter template is a repository with two key components:

**1. `cookiecutter.json`** — defines the variables and their defaults:

```json
{
  "repo_name": "repo_name",
  "project_name": "project_name",
  "author_name": "Your name",
  "python_version": "3.12",
  "open_source_license": ["No license file", "MIT", "BSD-3-Clause"]
}
```

List values (like `open_source_license`) become multiple-choice prompts. The first item is the default.

**2. A template directory** named `{{ cookiecutter.repo_name }}` — this is the actual project content. Any file or folder inside can reference variables using `{{ cookiecutter.variable_name }}` syntax, including in filenames and file contents.

For example, `pyproject.toml` inside the template might contain:

```toml
[project]
name = "{{ cookiecutter.repo_name }}"
description = "{{ cookiecutter.description }}"
requires-python = ">={{ cookiecutter.python_version }}"
```

When you run cookiecutter, all those placeholders are replaced with your answers.

## Re-using a Template

You can run cookiecutter from the same template as many times as you like to start new projects. Each run prompts you fresh and generates a clean new directory.

If you find yourself tweaking the generated output the same way every time, consider forking the template and making those changes there instead — that way future projects start closer to what you actually want.

!!! note
    Cookiecutter caches templates locally after the first download. To force a fresh download from the remote, use the `--no-input` flag with `--overwrite-if-exists`, or simply pass the URL again — it will prompt whether to re-download.

!!! example "Exercise"
    1. Install uv by following the [official instructions](https://docs.astral.sh/uv/getting-started/installation/).

    2. Generate a new project using the course template:

        ```bash
        cookiecutter https://github.com/yahei-DTU/project-template
        ```

    3. Navigate into the generated project and install the base dependencies:

        ```bash
        cd <your-repo-name>
        uv sync
        ```

    4. Put the project under version control and push it to a new, empty GitHub repository, following
        [Starting a New Project and Repository](#starting-a-new-project-and-repository).

    5. Add a new package to the project — for example, scikit-learn:

        ```bash
        uv add scikit-learn
        ```

    6. Verify the install worked by running a quick check:

        ```bash
        uv run python -c "import sklearn; print(sklearn.__version__)"
        ```

    7. Open `pyproject.toml` and confirm that `scikit-learn` now appears in the `dependencies` list.

    8. Commit the change and push it to GitHub:

        ```bash
        git add pyproject.toml uv.lock
        git commit -m "Add scikit-learn"
        git push
        ```
