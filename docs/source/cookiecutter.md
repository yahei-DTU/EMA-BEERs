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

    4. Add a new package to the project — for example, sklearn:

        ```bash
        uv add sklearn
        ```

    5. Verify the install worked by running a quick check:

        ```bash
        uv run python -c "import sklearn; print(sklearn.__version__)"
        ```

    6. Open `pyproject.toml` and confirm that `sklearn` now appears in the `dependencies` list.
