# radas Quick Examples

## Example 1: Simple Grid Search

```python
from radas import run_experiment
from ray import tune

def trainable(config):
    return {"score": config["a"] + config["b"]}

results = await run_experiment(
    user_name="your_username",
    experiment_name="grid-search-example",
    trainable=trainable,
    param_space={
        "a": tune.grid_search([1, 2, 3]),
        "b": tune.grid_search([10, 20]),
    },
    run_choice="run",  # CRITICAL for non-interactive
)
print(results["df"])
```

## Example 2: With Plotting

```python
import seaborn as sns
from radas import run_experiment
from ray import tune

def trainable(config):
    return {"accuracy": config["lr"] * config["epochs"]}

results = await run_experiment(
    user_name="your_username",
    experiment_name="plot-example",
    trainable=trainable,
    param_space={
        "lr": tune.grid_search([0.01, 0.1, 1.0]),
        "epochs": tune.grid_search([10, 50, 100]),
    },
    run_choice="run",  # CRITICAL for non-interactive
    plot_fn=sns.heatmap,
    plot_kwargs=dict(
        data=lambda df: df.pivot(index="config/lr", columns="config/epochs", values="accuracy"),
    ),
)
```

## Example 3: Running on Cluster

```python
# trainable.py (MUST be in separate file)
def trainable(config):
    return {"score": config["a"] * 2}
```

```python
# main.py or notebook
from radas import run_experiment
import ray.tune as tune
from trainable import trainable  # MUST import from separate file

results = await run_experiment(
    user_name="your_username",
    experiment_name="cluster-experiment",
    param_space=dict(
        a=tune.grid_search([1, 2]),
    ),
    trainable=trainable,
    run_with="cluster:atol-gpu-5090",
    run_choice="run",  # CRITICAL for non-interactive
)
```

## Example 4: Iterative Training with Early Stopping

```python
from ray.air import session
from ray.tune.schedulers import ASHAScheduler
from radas import run_experiment
from ray import tune

def trainable(config):
    for epoch in range(config["max_epochs"]):
        loss = 1.0 / (epoch + 1) * config["lr"]
        session.report({"loss": loss, "epoch": epoch})

results = await run_experiment(
    user_name="your_username",
    experiment_name="early-stopping-example",
    trainable=trainable,
    param_space={
        "lr": tune.loguniform(1e-4, 1e-1),
        "max_epochs": 100,
    },
    run_choice="run",  # CRITICAL for non-interactive
    tuner_init_kwargs=dict(
        tune_config=tune.TuneConfig(
            num_samples=20,
            metric="loss",
            mode="min",
            scheduler=ASHAScheduler(
                grace_period=5,
                max_t=100,
            ),
        ),
    ),
)
```

## Example 5: Custom DataFrame Processing

```python
import seaborn as sns
from radas import run_experiment
from ray import tune

def trainable(config):
    return {"loss": 1.0 / config["a"], "accuracy": config["a"] * 10}

def process_df_fn(df):
    # Add computed columns
    df["efficiency"] = df["accuracy"] / df["loss"]
    # Filter rows
    df = df[df["accuracy"] > 15]
    return df

results = await run_experiment(
    user_name="your_username",
    experiment_name="process-df-example",
    trainable=trainable,
    param_space={
        "a": tune.grid_search([1, 2, 3, 4, 5]),
    },
    run_choice="run",  # CRITICAL for non-interactive
    process_df_fn=process_df_fn,
    plot_fn=sns.barplot,
    plot_kwargs=dict(x="config/a", y="efficiency"),
)
```

## Example 6: Re-analyze Previous Results

```python
from radas import run_experiment
from ray import tune
import seaborn as sns

# First run the experiment
results = await run_experiment(
    user_name="your_username",
    experiment_name="reanalyze-example",
    trainable=trainable,
    param_space={"a": tune.grid_search([1, 2])},
    run_choice="run",  # CRITICAL for non-interactive
)

# Later, re-analyze with different visualization
results = await run_experiment(
    user_name="your_username",
    experiment_name="reanalyze-example",  # Same name
    trainable=trainable,
    param_space={"a": tune.grid_search([1, 2])},
    run_choice="run",  # CRITICAL for non-interactive
    dos=["analyze"],  # Only analyze, don't re-run
    plot_fn=sns.violinplot,
    plot_kwargs=dict(x="config/a", y="score"),
)
```

## Example 7: Using Optuna Search

```python
from ray.tune.search.optuna import OptunaSearch
from radas import run_experiment
from ray import tune

def trainable(config):
    # Objective to minimize
    return {"loss": (config["x"] - 2) ** 2 + (config["y"] - 3) ** 2}

results = await run_experiment(
    user_name="your_username",
    experiment_name="optuna-example",
    trainable=trainable,
    param_space={
        "x": tune.uniform(-10, 10),
        "y": tune.uniform(-10, 10),
    },
    run_choice="run",  # CRITICAL for non-interactive
    tuner_init_kwargs=dict(
        tune_config=tune.TuneConfig(
            search_alg=OptunaSearch(),
            num_samples=50,
            metric="loss",
            mode="min",
        ),
    ),
    runtime_env={"pip": ["optuna"]},
)

# Get best result
best = results["tuner"].get_results().get_best_result()
print(f"Best config: x={best.config['x']:.3f}, y={best.config['y']:.3f}")
print(f"Best loss: {best.metrics['loss']:.6f}")
```

## Example 8: Saving Artifacts in Trainable

```python
def trainable(config):
    import radas.cloud_artifact as rda

    rda.init(user_name="your_username")

    # Your training logic
    model_weights = {"layer1": [1, 2, 3]}

    # Save artifact
    rda.save_cloud_artifact(
        f"models/{config['experiment_name']}/weights.pkl",
        model_weights
    )

    return {"score": sum(model_weights["layer1"])}
```

## Example 9: Email Notification on Completion

```python
from radas import run_experiment
from ray import tune

results = await run_experiment(
    user_name="your_username",
    experiment_name="email-notify-example",
    trainable=trainable,
    param_space={"a": tune.grid_search([1, 2, 3])},
    run_choice="run",  # CRITICAL for non-interactive
    email_notification_to_address="your.email@example.com",
)
```

## Example 10: Accessing and Saving Figures

```python
import seaborn as sns
from ray import tune
from radas import run_experiment

def trainable(config):
    score = config["a"] + config["b"]
    return {"score": score}

config = {
    "a": tune.grid_search([1, 2]),
    "b": tune.grid_search([3, 4]),
}

results = await run_experiment(
    user_name="your_username",
    experiment_name="fig-example",
    trainable=trainable,
    param_space=config,
    run_choice="run",  # CRITICAL for non-interactive
    plot_fn=sns.lineplot,
    plot_kwargs=dict(
        x="config/a",
        y="score",
        hue="config/b",
    ),
)

# Access the figure object from results
fig = results["fig"]

# Save the figure to a file
fig.savefig("fig.png")
```

## Example 11: Non-Interactive Cluster Experiment

```python
from radas import run_experiment
import ray.tune as tune
from trainable import trainable  # MUST import from separate file

results = await run_experiment(
    user_name="your_username",
    experiment_name="non-interactive-cluster",
    param_space=dict(
        a=tune.grid_search([1, 2]),
    ),
    trainable=trainable,
    run_with="cluster:atol-gpu-5090",
    run_choice="run",  # CRITICAL for non-interactive
)
```
