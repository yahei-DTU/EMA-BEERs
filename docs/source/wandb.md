# Experiment Tracking with W&B

When training machine learning models you quickly accumulate runs: different hyperparameters, different datasets, different architectures. Keeping track of what configuration produced what result in a spreadsheet or in your head does not scale. **Weights & Biases (W&B)** is a platform that automatically logs metrics, hyperparameters, and artifacts from your training runs and lets you compare them in an interactive dashboard.

## What W&B Solves

Without experiment tracking, a typical training loop just prints to the terminal:

```python
for epoch in range(epochs):
    loss = train_one_epoch(model, dataloader)
    print(f"Epoch {epoch}: loss={loss:.4f}")
```

You lose that output the moment the terminal closes, and comparing runs means mentally replaying what you changed. With W&B, every run is logged automatically to a web dashboard where you can plot, filter, and compare across experiments.

## Installing W&B

```bash
uv add wandb
```

## Getting Started

W&B requires a free account. Create one at [wandb.ai](https://wandb.ai), then log in once from the terminal:

```bash
uv run wandb login
```

This stores an API key locally so you do not need to log in again.

Alternatively, you can store your API key in the project's `.env` file. This is useful when running in environments where interactive login is not possible (e.g. CI, Docker, remote servers):

```bash
# .env
WANDB_API_KEY=your_api_key_here
```

W&B automatically picks up `WANDB_API_KEY` from the environment. You can find your key at [wandb.ai/settings](https://wandb.ai/settings).

!!! warning
    Never commit `.env` to Git. It is already listed in `.gitignore` in this project, but double-check before pushing. Treat your API key like a password.

## Basic Usage

The core workflow is three lines: initialise a run, log metrics each step, and finish.

Call `load_dotenv()` at the top of your script to load `WANDB_API_KEY` (and any other variables) from `.env` before W&B initialises:

```python
import wandb
from dotenv import load_dotenv

load_dotenv()  # W&B automatically picks up WANDB_PROJECT, WANDB_ENTITY and WANDB_API_KEY

wandb.init(config={"lr": 0.001, "batch_size": 32, "epochs": 10})

for epoch in range(wandb.config.epochs):
    loss = train_one_epoch(model, dataloader)
    val_loss = evaluate(model, val_dataloader)

    wandb.log({"train/loss": loss, "val/loss": val_loss, "epoch": epoch})

wandb.finish()
```

After the first `wandb.init`, a link to the run's dashboard is printed in the terminal. Every `wandb.log` call sends a data point to that dashboard in real time.

## Integrating with Hydra

Since the project uses Hydra for configuration, you can pass the Hydra config directly to `wandb.init` so every run is fully described by its config:

```python
import wandb
import hydra
from omegaconf import DictConfig, OmegaConf

@hydra.main(config_path="../../configs", config_name="config_dev", version_base=None)
def train(cfg: DictConfig) -> None:
    wandb.init(
        config=OmegaConf.to_container(cfg, resolve=True),
    )

    for epoch in range(cfg.experiments.epochs):
        loss = train_one_epoch(model, dataloader)
        wandb.log({"train/loss": loss, "epoch": epoch})

    wandb.finish()
```

`OmegaConf.to_container` converts the Hydra config to a plain dictionary that W&B can serialize.

## What You Can Log

`wandb.log` accepts any scalar, but also richer types:

```python
# Scalars
wandb.log({"loss": 0.42, "accuracy": 0.91})

# Images (e.g. predictions vs ground truth)
wandb.log({"predictions": [wandb.Image(img) for img in sample_images]})

# Matplotlib figures
import matplotlib.pyplot as plt
fig, ax = plt.subplots()
ax.plot(losses)
wandb.log({"loss_curve": wandb.Image(fig)})
plt.close(fig)
```

## Saving Model Artifacts

W&B can store trained model files so you can retrieve any checkpoint later:

```python
artifact = wandb.Artifact("model", type="model")
artifact.add_file("models/best_model.pt")
wandb.log_artifact(artifact)
```

Artifacts are versioned automatically. You can download a specific version later with:

```python
artifact = wandb.use_artifact("model:v3")
artifact.download()
```

## The W&B Dashboard

Once a run is logged, the dashboard at [wandb.ai](https://wandb.ai) lets you:

- **Plot** metrics over time across multiple runs
- **Compare** runs side by side by any config parameter
- **Filter** runs by metric (e.g. show only runs where val_loss < 0.3)
- **Group** runs by hyperparameter to see trends

This is particularly useful when combined with Hydra's `--multirun` flag — you can sweep over learning rates or batch sizes and immediately see the results compared in W&B.

!!! note
    By default W&B logs runs to the cloud. If you need to work offline, set the environment variable `WANDB_MODE=offline` before running your script. Runs will be saved locally and can be synced later with `wandb sync`.

## Exercise

!!! example "Exercise"
    1. Create a free account at [wandb.ai](https://wandb.ai) and log in:

        ```bash
        uv run wandb login
        ```

    2. Add `wandb` to the project:

        ```bash
        uv add wandb
        ```

    3. In `src/ema_beers/data.py`, extend the `main()` function you wrote in the Hydra exercise to initialise a W&B run and log summary statistics of the generated dataset:

        ```python
        import wandb
        from dotenv import load_dotenv

        load_dotenv()

        wandb.init(config=OmegaConf.to_container(cfg, resolve=True))

        df = generate_data(cfg.experiments.n_samples)

        wandb.log({
            "n_samples": len(df),
            "abv_mean": df["abv"].mean(),
            "abv_std": df["abv"].std(),
            "noise_mean": (df["abv"] - df["abv_true"]).mean(),
            "noise_std": (df["abv"] - df["abv_true"]).std(),
        })

        wandb.finish()
        ```

    4. Run the script and follow the link printed in the terminal to see your run in the dashboard:

        ```bash
        uv run python -m ema_beers.data
        ```

    5. Run it again with a different `n_samples` and compare the two runs in the W&B UI:

        ```bash
        uv run python -m ema_beers.data experiments.n_samples=500
        ```

    6. **Bonus:** log a scatter plot of measured vs. true ABV as a W&B figure so you can visually inspect the noise across runs:

        ```python
        import matplotlib.pyplot as plt

        fig, ax = plt.subplots()
        ax.scatter(df["abv_true"], df["abv"], alpha=0.4, s=10)
        ax.plot([df["abv_true"].min(), df["abv_true"].max()],
                [df["abv_true"].min(), df["abv_true"].max()], "r--")
        ax.set_xlabel("True ABV")
        ax.set_ylabel("Measured ABV")
        wandb.log({"abv_scatter": wandb.Image(fig)})
        plt.close(fig)
        ```
