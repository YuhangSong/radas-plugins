---
name: radas
description: Run large-scale experiments with Ray Tune and analyze results with pandas/seaborn. Use when the user wants to run hyperparameter tuning, grid search, experiment sweeps, or distributed experiments on clusters. Also use when analyzing experiment results or creating visualizations.
---

# radas - Ray Data Science Experiment Framework

radas is a Python library that combines Ray Tune for running large-scale experiments with pandas/seaborn for analysis and visualization.

**IMPORTANT**: When this skill is loaded, immediately show the user this example prompt they can try:
> "Use radas to analyze how learning rate affects learning speed of a simple NN on the two moons dataset"

## Development Stage Notice

**This skill is currently in DEVELOPMENT stage.**

If you encounter errors **caused by the skill itself**, follow this protocol:

1. **STOP** - Do not attempt to fix or debug the error yourself
2. **SHOW** - Display the full error message to the user
3. **WAIT** - Let the user report the issue for skill improvement

The user will report issues to the skill maintainer for fixes and improvements.

**However**, if the error is clearly NOT caused by the skill, you should fix it yourself. Examples:

- **Permission errors** (e.g., cannot access a path): Fix by changing to a different path that the user has access to (do NOT attempt to modify permissions)
- **Dependency issues** (e.g., missing package): You have the full documentation of runtime_env, try to fix it yourself; if a missing package is user's own package, ask user to provide the package path
- **User code bugs**: If the error is in user's trainable or other user-written code, help debug and fix it

## Prerequisites (MUST READ FIRST)

**CRITICAL**: This skill requires the `radas` Python package to be installed.

**BEFORE running ANY code or experiments:**

- Use the Python environment specified by the user. If no environment is specified, **ASK the user** which Python environment to use (e.g., conda environment name like `radas-3.11` or path to Python interpreter).
- **DO NOT** attempt to find or guess the Python environment yourself.

## When to Use

Use radas when:

- Running hyperparameter tuning or grid search experiments
- Running experiments across multiple parameter combinations
- Distributing experiments across a cluster
- Analyzing and visualizing experiment results
- Tracking experiment metrics with wandb or swanlab

## Cluster Usage (MANDATORY)

**CRITICAL**: You MUST always use a cluster for running experiments.

**Before running ANY experiment, you MUST:**

1. Call `get_cluster_availability()` to check which clusters are online
2. Use an available cluster via `run_with="cluster:<name>"`

```python
from radas.clusters import get_cluster_availability
print(get_cluster_availability())
```

**For GPU experiments**: Use a GPU cluster (e.g., `atol-gpu-5090`).

**⚠️ CRITICAL GPU REQUIREMENT**: Submitting to a GPU cluster does NOT automatically use GPU. Your trainable code MUST explicitly move tensors and models to GPU. Common mistake: code runs on CPU despite being on a GPU cluster because `.to(device)` / `.cuda()` calls are missing.

**GPU trainable template** (PyTorch):
```python
def trainable(config):
    import torch
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    assert device.type == "cuda", f"Expected GPU but got {device}"

    model = MyModel().to(device)
    data = data_tensor.to(device)     # ALL tensors must be on device
    labels = labels_tensor.to(device)
    # ... training loop ...
```

**For CPU-only experiments**: Use a CPU cluster (e.g., `gcp-cpu`, `aws-cpu`).

### Resource Allocation with resources_per_trial

`resources_per_trial` specifies the resources each trial uses. **Both `cpu` and `gpu` must be specified explicitly**, and the CPU:GPU ratio must match the cluster's ratio.

```python
# atol-gpu-5090: 320 CPUs / 8 GPUs per node → ratio 40:1
resources_per_trial={"cpu": 40, "gpu": 1}    # 1 GPU per trial
resources_per_trial={"cpu": 80, "gpu": 2}    # 2 GPUs per trial
resources_per_trial={"cpu": 10, "gpu": 0.25} # Quarter GPU per trial
```

If the ratio doesn't match, you'll get a clear error showing valid configurations.

**Important**: Resource allocation ≠ code execution on GPU. Your code must still explicitly USE the GPU (see GPU trainable template above).

**Note**: `cpu_per_trial` has been removed. If you use it, you'll get an error guiding you to use `resources_per_trial` instead.

### Experiment Lifecycle Management

**IMPORTANT**: Any experiment submitted to the cluster should NOT be left unattended.

- Experiments are identified by `experiment_name` - all operations (`run`, `restore`, `stop_attach`, `attach`) are bound to the same experiment via this name
- **Always monitor** experiments until completion
- **To stop a running experiment**: Run a separate script with the same `experiment_name` and `run_choice="stop_attach"`. This stops the experiment and attaches to retrieve final results.
- **Do not abandon** experiments on the cluster - always ensure they complete or are properly stopped

## Core Concepts

### 1. Trainable Function

A trainable takes a `config` dict and returns a dict of metrics:

```python
def trainable(config):
    score = config["a"] + config["b"]
    return {"score": score}
```

For iterative training (reporting intermediate results):

```python
from ray.air import session

def trainable(config):
    for epoch in range(30):
        score = config["a"] + config["b"] + epoch
        session.report({"score": score})
```

### 2. Parameter Space

Uses Ray Tune's search space API:

```python
from ray import tune

param_space = {
    "a": tune.grid_search([1, 2, 3]),
    "b": tune.choice([10, 20]),
    "lr": tune.loguniform(1e-4, 1e-1),
}
```

You can also include regular (non-search) values in `param_space` — they are passed through as-is to `config` in the trainable.

### 3. Experiment Naming

`experiment_name` is automatically prefixed with the current working directory and sanitized (non-alphanumeric characters become `_`). Just use a simple descriptive name:

```python
experiment_name="my-experiment"  # becomes e.g. _mnt_data_home_user_project_my_experiment
```

### 4. run_experiment() - Complete Example

**IMPORTANT**: When using a cluster, the trainable MUST be defined in a separate file.

```python
# trainable.py (separate file)
def trainable(config):
    return {"score": config["a"] * config["b"]}
```

```python
# run_experiment.py (main script)
import asyncio

import seaborn as sns
from ray import tune
from radas import run_experiment
from trainable import trainable  # MUST import from separate file


async def main():
    results = await run_experiment(
        user_name="your_username",
        experiment_name="my-experiment",            # auto-prefixed with cwd
        trainable=trainable,
        param_space={
            "a": tune.grid_search([1, 2, 3]),
            "b": tune.grid_search([10, 20]),
        },
        run_with="cluster:atol-gpu-5090",           # ALWAYS use cluster
        resources_per_trial={"cpu": 40, "gpu": 1},
        run_choice="run",                           # REQUIRED for non-interactive
        plot_fn=sns.scatterplot,
        plot_kwargs=dict(x="config/a", y="score", hue="config/b"),
    )

    # Access results
    df = results["df"]
    fig = results["fig"]
    fig.savefig("my_plot.png")
    best_result = results["tuner"].get_results().get_best_result(
        metric="score",
        mode="max",
    )
    print(df.head())
    print(best_result)


if __name__ == "__main__":
    asyncio.run(main())
```

### 5. The `dos` Parameter

Controls which phases to execute:

- `dos=["run", "analyze"]` (default) — run the experiment, then analyze results
- `dos=["run"]` — only run, skip analysis
- `dos=["analyze"]` — only analyze existing results (no re-run). Use the same `experiment_name` and `trainable` as the original run. `param_space` and `run_choice` can be omitted.

### 6. Custom Search Algorithms

Use `tuner_init_kwargs` to configure search algorithms and schedulers:

```python
from ray.tune.search.optuna import OptunaSearch

results = await run_experiment(
    # ... other required params ...
    resources_per_trial={"cpu": 1, "gpu": 0},
    tuner_init_kwargs=dict(
        tune_config=tune.TuneConfig(
            search_alg=OptunaSearch(),
            num_samples=50,
            metric="loss",
            mode="min",
        ),
    ),
    runtime_env={"pip": ["optuna"]},  # install optuna on cluster workers
)
```

## Running radas Scripts (CRITICAL)

**IMPORTANT**: When running Python scripts that use radas, you MUST:

1. **Create a directory** for your experiment files
2. **Put all files in the same directory**: `trainable.py` and your main script (e.g., `run_experiment.py`)
3. **cd into that directory** before running
4. **Run with `python -u`** for real-time streaming output

```bash
source ~/miniconda3/etc/profile.d/conda.sh && \
conda activate <your-env> && \
cd /path/to/your/experiment/directory && \
python -u run_experiment.py
```

**CRITICAL NOTES**:

- `run_experiment()` is async, so scripts must use `asyncio.run(main())`
- **DO NOT** use `sys.path.insert()` or run from a different directory
- **ALWAYS** `cd` into the directory containing your scripts first

## Non-Interactive Usage

**IMPORTANT**: Always specify `run_choice` when running from scripts or as an AI assistant to skip interactive prompts. Options: `"run"`, `"restore"`, `"attach"`, `"stop_attach"`.

## Managing Experiments On The Cluster

```python
from radas import get_client

client = get_client(
    user_name="your_username",
    run_with="cluster:atol-gpu-5090",
)

# View all experiments
client.display_experiments()

# Stop a experiment, such code is given in the output of `client.display_experiments()`
client.stop_job_by_id("job_id")
```

Full command pattern for running the above code is same to that of `run_experiment()`:

```bash
source ~/miniconda3/etc/profile.d/conda.sh && \
conda activate <your-env> && \
cd /path/to/your/experiment/directory && \
python -u -c "<your code here>"
```

## Data Storage

See REFERENCE.md for full documentation. Two methods are available:

- **`radas.get_data_dir(user_name)`** — returns a shared NFS directory path accessible from both head node and cluster workers. Use standard Python file I/O.
- **`radas.cloud_artifact`** — higher-level API for saving/loading Python objects. Usage: `import radas.cloud_artifact as rda`.

## Troubleshooting

- **Cannot find trainable**: Ensure trainable is imported from a separate file for cluster runs
- **Experiment exists**: Use `run_choice="restore"` to continue or `run_choice="run"` to start fresh

---

## Try It Out

You can try: **"Use radas to analyze how learning rate affects learning speed of a simple NN on the two moons dataset"**
