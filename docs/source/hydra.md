# Configuration Management with Hydra

As a machine learning project grows, you accumulate a lot of settings: dataset paths, model hyperparameters, training schedules, logging levels. Hardcoding these values in your scripts, or scattering them across multiple ad-hoc YAML files, quickly becomes unmanageable. **Hydra** is a framework that solves this by giving you a structured, composable configuration system with a clean Python API.

## What Problem Hydra Solves

Without a configuration system, a typical training script ends up with values hardcoded at the top, or a tangle of `argparse` arguments:

```python
# Without Hydra — hard to maintain and reproduce
LEARNING_RATE = 0.001
BATCH_SIZE = 32
DATA_PATH = "data/train.csv"
```

With Hydra, all of this lives in YAML files that your script loads automatically. You can swap out configurations from the command line without touching the code, compose different config groups together, and every run logs exactly what configuration it used.

## Installing Hydra

```bash
uv add hydra-core
```

## How Hydra Works

Hydra loads configuration from YAML files and injects it into your Python functions as a structured config object. The entry point is the `@hydra.main` decorator.

A minimal example:

```python
import hydra
from omegaconf import DictConfig

@hydra.main(config_path="configs", config_name="config_dev", version_base=None)
def train(cfg: DictConfig) -> None:
    print(f"Learning rate: {cfg.experiments.lr}")
    print(f"Batch size: {cfg.experiments.batch_size}")
    print(f"Seed: {cfg.seed}")

if __name__ == "__main__":
    train()
```

When you run this script, Hydra finds `configs/config_dev.yaml`, loads it (including any composed sub-configs), and passes the result as `cfg`.

## Config Files and Composition

The real power of Hydra is **config composition**. Instead of one large config file per environment, you split settings into groups and compose them together.

The project is structured like this:

```
configs/
    config_dev.yaml       <- development config
    config_prod.yaml      <- production config
    config_test.yaml      <- test config
    datasets/
        data.yaml         <- dataset settings
    models/
        model.yaml        <- model architecture settings
    experiments/
        exp.yaml          <- hyperparameters
```

The top-level config files use a `defaults` list to pull in the sub-configs:

```yaml
# configs/config_dev.yaml
defaults:
  - datasets: data      # loads configs/datasets/data.yaml
  - models: model       # loads configs/models/model.yaml
  - experiments: exp    # loads configs/experiments/exp.yaml
  - _self_              # values in this file override the above

seed: 42

hydra:
  job_logging:
    version: 1
    root:
      level: DEBUG
      handlers: [console]
```

The `_self_` entry controls where the values in the current file are merged relative to the defaults. Placing it last means local values take precedence.

## Overriding Values from the Command Line

Any config value can be overridden at runtime without touching the files:

```bash
# Override a single value
uv run python src/ema_beers/train.py seed=123

# Override a nested value
uv run python src/ema_beers/train.py experiments.lr=0.01

# Swap an entire config group
uv run python src/ema_beers/train.py --config-name=config_prod
```

This makes it straightforward to run experiments with different hyperparameters without duplicating files or editing code.

## Multirun: Sweeping Over Parameters

Hydra has built-in support for running the same script across multiple configurations with the `--multirun` flag:

```bash
uv run python src/ema_beers/train.py --multirun experiments.lr=0.001,0.01,0.1
```

This launches three separate runs, one for each learning rate. Each run gets its own output directory under `outputs/`.

## Output Directory

By default, Hydra creates an `outputs/` directory and saves a copy of the full resolved config for every run:

```
outputs/
    2024-05-01/
        14-32-05/
            .hydra/
                config.yaml    <- exact config used for this run
                overrides.yaml <- any command-line overrides
            train.log
```

This makes every experiment fully reproducible: you can always look up exactly what configuration produced a given result.

!!! note
    Add `outputs/` to your `.gitignore`. These directories can grow large quickly and the configs inside are already reproducible from your YAML files and override history.

## Exercise

!!! example "Exercise"
    1. Add `hydra-core` to the project:

        ```bash
        uv add hydra-core
        ```

    2. Add some values to `configs/experiments/exp.yaml`, for example a sample size:

        ```yaml
        n_samples: 200
        ```

    3. In `data.py`, copy the following function and complete it:

        The function simulates brewery hydrometer measurements. A brewer measures the sugar content of wort before and after fermentation using a hydrometer. The difference between **original gravity (OG)** and **final gravity (FG)** tells you how much sugar was converted to alcohol. The standard formula to convert that difference to **alcohol by volume (ABV)** is:

        $$\text{ABV} = (\text{OG} - \text{FG}) \times 131.25$$

        In practice, a hydrometer reading has small measurement noise — so `abv` is the noisy measured value and `abv_true` is what the ABV actually is based on the true gravities.
    
        ```python
        import numpy as np
        import pandas as pd

        def generate_data(n):
            og = np.random.uniform(1.040, 1.090, n)   # Original gravity
            fg = og - np.random.uniform(0.008, 0.020, n)  # Final gravity (fermentation drops it)

            abv_true = (og - fg) * 131.25
            abv = abv_true + np.random.normal(0, 0.15, n)  # Small measurement noise

            return pd.DataFrame({"abv": abv, "abv_true": abv_true})
        ```

    Now write a function `main()` that uses `@hydra.main` to load `config_dev`, generates the data with `n_samples` from the experiment settings, and prints the first few rows.

    4. Run the script and confirm the output looks reasonable (ABV values between ~3% and ~15%):

        ```bash
        uv run python -m ema_beers.data
        ```

    5. Run it with a different value for `n_samples` in `exp.yaml`. You can also override the number of samples from the command line without editing any file:

        ```bash
        uv run python -m ema_beers.data experiments.n_samples=50
        ```

    6. Check the `outputs/` directory and inspect the saved `.hydra/config.yaml` to see the full resolved config.
