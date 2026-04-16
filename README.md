# EMA-BEERs

Introduction to course in EMA BEERs

## Project structure

The directory structure of the project looks like this:
```txt
├── .devcontainer/            # Dev container configuration
│   ├── devcontainer.json
│   └── post_create.sh
├── .github/                  # Github actions and dependabot
│   ├── dependabot.yaml
│   └── workflows/
│       ├── data-tests.yaml
│       ├── deploy_docs.yaml
│       ├── linting.yaml
│       ├── pre-commit-update.yaml
│       └── tests.yaml
├── configs/                  # Configuration files
│   ├── datasets/
│   │   └── data.yaml
│   ├── experiments/
│   │   └── exp.yaml
│   ├── models/
│   │   └── model.yaml
│   ├── config_dev.yaml
│   ├── config_prod.yaml
│   └── config_test.yaml
├── dockerfiles/              # Dockerfiles
│   ├── api.dockerfile
│   └── train.dockerfile
├── docs/                     # Documentation
│   ├── source/
│   │   └── index.md
│   ├── mkdocs.yaml
│   └── README.md
├── models/                   # Trained models
├── notebooks/                # Jupyter notebooks
├── reports/                  # Reports
│   └── figures/
├── src/                      # Source code
│   └── project_name/
│       ├── utils/
│       │   └── __init__.py
│       ├── __init__.py
│       ├── api.py
│       ├── data.py
│       ├── evaluate.py
│       ├── model.py
│       ├── train.py
│       └── visualize.py
├── tests/                    # Tests
│   ├── __init__.py
│   ├── test_api.py
│   ├── test_data.py
│   └── test_model.py
├── .env
├── .gitignore
├── .pre-commit-config.yaml
├── .python-version
├── AGENTS.md
├── LICENSE
├── pyproject.toml            # Python project file
├── README.md                 # Project README
├── requirements.txt          # Production dependencies
├── requirements_dev.txt      # Development dependencies
└── tasks.py                  # Project tasks
```


Template based on [mlops_template](https://github.com/SkafteNicki/mlops_template),
a [cookiecutter template](https://github.com/cookiecutter/cookiecutter).
