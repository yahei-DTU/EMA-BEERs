# Documentation with MkDocs

Good documentation is what makes a project usable by others — and by your future self. Writing it should be as frictionless as possible. **MkDocs** is a static site generator that turns Markdown files into a clean, searchable documentation website. This project uses the **Material for MkDocs** theme, which adds a polished look, dark mode, admonitions, and many other features on top of the base tool.

## What is MkDocs?

MkDocs takes a folder of Markdown files and a single `mkdocs.yaml` configuration file and generates a complete HTML site. You write plain Markdown, run one command, and get a professional documentation site.

The generated site can be hosted anywhere that serves static files — GitHub Pages, Netlify, S3 — or browsed locally during development.

## Installing MkDocs

MkDocs and the Material theme are already in the project's dev dependencies. After `uv sync` they are available immediately:

```bash
uv add mkdocs mkdocs-material mkdocstrings-python
```

## Project Structure

The documentation lives in `docs/`:

```
docs/
    mkdocs.yaml       <- configuration file
    source/           <- all Markdown pages go here
        index.md
        uv.md
        hydra.md
        ...
```

`mkdocs.yaml` uses `docs_dir: source` to point at the `source/` folder as the root for all pages.

## The mkdocs.yaml File

All configuration lives in `mkdocs.yaml`. The project's current configuration:

```yaml
site_name: EMA-BEERs
site_author: yahei-DTU
docs_dir: source

theme:
  name: material
  features:
    - content.code.copy
    - content.code.annotate
  palette:
    - scheme: default
      toggle:
        icon: material/brightness-7
        name: Switch to dark mode
    - scheme: slate
      toggle:
        icon: material/brightness-4
        name: Switch to light mode

markdown_extensions:
  - admonition
  - pymdownx.details
  - pymdownx.superfences

plugins:
  - search
  - mkdocstrings:
      handlers:
        python:
          options:
            show_root_heading: true
            separate_signature: true
            show_signature_annotations: true

nav:
  - Home: index.md
  - Content:
    - uv: uv.md
    - Cookiecutter: cookiecutter.md
    ...
```

The key sections are:

- **`theme`** — the Material theme with dark/light mode toggle and code copy buttons
- **`markdown_extensions`** — extra Markdown features like admonitions (`!!! note`) and code blocks inside admonitions
- **`plugins`** — `search` adds a search bar; `mkdocstrings` auto-generates API docs from docstrings
- **`nav`** — defines the structure and order of the sidebar

## Serving Locally

During development, run:

```bash
uv run mkdocs serve -f docs/mkdocs.yaml
```

This starts a local server at `http://127.0.0.1:8000` that **live-reloads** whenever you save a Markdown file — no need to refresh manually.

## Writing Pages

Each page is a Markdown file in `docs/source/`. Add it to the `nav` in `mkdocs.yaml` to make it appear in the sidebar:

```yaml
nav:
  - Home: index.md
  - My New Page: my_page.md
```

Pages can be grouped into sections:

```yaml
nav:
  - Home: index.md
  - Content:
    - uv: uv.md
    - Hydra: hydra.md
  - Extras:
    - Docker: docker.md
```

## Admonitions

The Material theme supports styled call-out boxes called admonitions. The project has `admonition`, `pymdownx.details`, and `pymdownx.superfences` enabled to support these:

```markdown
!!! note
    This is a note box.

!!! warning
    This is a warning box.

!!! example "Exercise"
    This is an exercise box.
```

The available types include `note`, `warning`, `tip`, `info`, `example`, `danger`, and more — each renders with a distinct colour and icon.

## Auto-generating API Docs with mkdocstrings

The `mkdocstrings` plugin can pull docstrings directly from your Python source code and render them as documentation pages. In any Markdown file, use:

```markdown
::: ema_beers.data
::: ema_beers.model
```

This renders the full API reference for those modules automatically — function signatures, type annotations, and docstrings — without duplicating anything.

## Deploying to GitHub Pages

The project's `deploy_docs.yaml` workflow deploys the site automatically on every push to `main`. To deploy manually:

```bash
uv run mkdocs gh-deploy --force -f docs/mkdocs.yaml
```

This builds the site and pushes it to the `gh-pages` branch. GitHub Pages serves it from there.

!!! warning
    GitHub Pages must be enabled in the repository settings before the site is served. Go to **Settings → Pages**, set the source to **Deploy from a branch**, and select `gh-pages`.

## Exercise

!!! example "Exercise"
    1. Start the local docs server and open it in the browser:

        ```bash
        uv run mkdocs serve -f docs/mkdocs.yaml
        ```

    2. Edit one of the existing Markdown pages and save it — observe the browser reload automatically.

    3. Create a new page `docs/source/my_page.md` and add it to the `nav` in `mkdocs.yaml`. Confirm it appears in the sidebar.

    4. Add an `!!! note` admonition to any page and verify it renders correctly.

    5. Add a Google-style docstring to a function in `src/ema_beers/data.py`, then add a `:::` reference to it in a Markdown page and check the rendered output.

    6. Push to `main` and verify the `deploy_docs.yaml` workflow deploys the updated site to GitHub Pages.
