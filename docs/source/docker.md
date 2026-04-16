# Containerisation with Docker

A common problem when sharing code is: *"it works on my machine"*. Different operating systems, Python versions, and installed packages mean that code working locally may fail on someone else's machine or on a server. **Docker** solves this by packaging your code together with everything it needs to run into a single, portable unit called a **container**.

## What is Docker?

A Docker **container** is a lightweight, isolated environment that runs the same way on any machine. Think of it as a stripped-down virtual machine — it has its own filesystem, its own Python installation, and its own dependencies — but it shares the host machine's kernel, making it much faster and smaller than a full VM.

A container is built from an **image**, which is defined by a **Dockerfile** — a text file with step-by-step instructions for assembling the environment.

## Installing Docker

Download and install Docker Desktop from [docs.docker.com/get-docker](https://docs.docker.com/get-docker/). Once installed, verify it works:

```bash
docker --version
```

## Dockerfile Basics

A Dockerfile is a recipe for building an image. Each line is an instruction:

```dockerfile
FROM python:3.12-slim          # start from an official Python image

WORKDIR /app                   # set the working directory inside the container

COPY pyproject.toml .          # copy files into the container
COPY src src/

RUN pip install .              # run commands to install dependencies

ENTRYPOINT ["python", "src/my_project/train.py"]  # command to run when the container starts
```

The key instructions are:

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image to build on top of |
| `WORKDIR` | Sets the working directory for subsequent commands |
| `COPY` | Copies files from your machine into the image |
| `RUN` | Executes a shell command during the build |
| `ENTRYPOINT` | The command that runs when the container starts |

## Dockerfiles in This Project

The project has two Dockerfiles in `dockerfiles/`:

### train.dockerfile — Training

Builds an image for running the training script:

```dockerfile
FROM ghcr.io/astral-sh/uv:python3.12-alpine AS base

COPY uv.lock uv.lock
COPY pyproject.toml pyproject.toml

RUN uv sync --frozen --no-install-project

COPY src src/
COPY README.md README.md
COPY LICENSE LICENSE

RUN uv sync --frozen

ENTRYPOINT ["uv", "run", "src/ema_beers/train.py"]
```

It uses the official `uv` base image and installs dependencies in two stages: first the dependencies alone (so Docker can cache this layer), then the project source. This means rebuilding after a code change does not re-download all packages.

### api.dockerfile — API Serving

Builds an image for running the FastAPI server:

```dockerfile
FROM ghcr.io/astral-sh/uv:python3.12-alpine AS base

COPY uv.lock uv.lock
COPY pyproject.toml pyproject.toml

RUN uv sync --frozen --no-install-project

COPY src src/
COPY README.md README.md
COPY LICENSE LICENSE

RUN uv sync --frozen

ENTRYPOINT ["uv", "run", "uvicorn", "src.ema_beers.api:app", "--host", "0.0.0.0", "--port", "8000"]
```

The only difference from the training image is the `ENTRYPOINT` — it starts the uvicorn web server instead of the training script.

## Building and Running

**Build an image:**

```bash
docker build -f dockerfiles/train.dockerfile -t ema-beers-train .
```

- `-f` specifies which Dockerfile to use
- `-t` gives the image a name (tag)
- `.` is the **build context** — the directory Docker has access to when copying files

**Run the training container:**

```bash
docker run ema-beers-train
```

**Run the API container and expose its port:**

```bash
docker build -f dockerfiles/api.dockerfile -t ema-beers-api .
docker run -p 8000:8000 ema-beers-api
```

`-p 8000:8000` maps port 8000 inside the container to port 8000 on your machine, so you can reach the API at `http://localhost:8000`.

## Layer Caching

Docker builds images layer by layer, and **caches each layer**. If a layer has not changed since the last build, Docker reuses the cached version instead of rebuilding it. This is why the Dockerfiles copy `pyproject.toml` and run `uv sync` before copying the source code:

```dockerfile
# These layers are cached as long as pyproject.toml/uv.lock don't change
COPY uv.lock uv.lock
COPY pyproject.toml pyproject.toml
RUN uv sync --frozen --no-install-project

# Only this layer is rebuilt when you change source code
COPY src src/
RUN uv sync --frozen
```

If you copy the source first and then install dependencies, every code change would trigger a full re-install of all packages — which can take minutes.

!!! note
    The build context (`.`) is sent to the Docker daemon on every build. Add a `.dockerignore` file to exclude large directories like `.venv`, `data/`, `models/`, and `outputs/` from being copied, which speeds up builds significantly.

## Dev Containers

The Dockerfiles above are for running training and the API. There is a second Docker-based workflow for **development itself**: the dev container.

A **dev container** is a Docker container that your editor runs inside of. Instead of installing Python, uv, and extensions locally, you open the project in a container that already has everything set up. The experience feels identical to working locally — you edit files, run tests, use the terminal — but the environment is defined in code and identical for everyone on the team.

This project's dev container is configured in `.devcontainer/`:

```
.devcontainer/
    devcontainer.json     <- tells VS Code how to build and use the container
    post_create.sh        <- runs once after the container is created
```

### devcontainer.json

```json
{
  "name": "devcontainer",
  "dockerFile": "devcontainer.dockerfile",
  "postCreateCommand": "bash .devcontainer/post_create.sh",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.vscode-pylance",
        "charliermarsh.ruff",
        "eamodio.gitlens",
        "github.copilot"
      ]
    }
  }
}
```

- `dockerFile` — the Dockerfile used to build the development environment
- `postCreateCommand` — a script that runs once after the container is first created
- `customizations.vscode.extensions` — VS Code extensions that are automatically installed inside the container

### post_create.sh

This script runs automatically after the container is created:

```bash
uv sync --locked          # install all dependencies
uv run pre-commit install # install git hooks
```

By the time the container is ready, the full environment is set up — dependencies installed, hooks active — without any manual steps.

### Opening the Dev Container

You need **VS Code** with the **Dev Containers** extension installed. Then:

1. Open the project folder in VS Code
2. When prompted *"Reopen in Container"*, click it — or open the Command Palette (`Ctrl+Shift+P`) and run **Dev Containers: Reopen in Container**
3. VS Code builds the image (first time only) and reopens inside the container
4. `post_create.sh` runs automatically

From this point your terminal, Python interpreter, and extensions all run inside the container. Anyone cloning the repo and opening it in VS Code gets an identical environment.

!!! note
    The dev container is for development — editing code, running tests, using the terminal. The `train.dockerfile` and `api.dockerfile` are for production use: packaging the training job or the API for deployment.

## Exercise

!!! example "Exercise"
    1. Install Docker Desktop and verify it is running:

        ```bash
        docker --version
        ```

    2. Build the training image:

        ```bash
        docker build -f dockerfiles/train.dockerfile -t ema-beers-train .
        ```

    3. Run the training container and observe the output:

        ```bash
        docker run ema-beers-train
        ```

    4. Build and run the API image, then open `http://localhost:8000/docs` in your browser:

        ```bash
        docker build -f dockerfiles/api.dockerfile -t ema-beers-api .
        docker run -p 8000:8000 ema-beers-api
        ```

    5. Make a small change to `src/ema_beers/train.py` and rebuild the training image. Notice which layers are rebuilt and which are served from cache.

    6. **Bonus:** create a `.dockerignore` file that excludes `.venv/`, `data/`, `models/`, and `outputs/`, then rebuild and compare the build time.
