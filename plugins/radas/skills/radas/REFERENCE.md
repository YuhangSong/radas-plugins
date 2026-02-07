# radas API Reference

## run_experiment() Full Signature

```python
async def run_experiment(
    # Required
    user_name: str,
    trainable: Callable,            # Required (always)
    experiment_name: str,           # Required (always)

    # Cluster
    run_with: str = "local",               # "local" or "cluster:<name>"
    runtime_env: dict = {},

    # Parameter Space
    param_space: dict = {},

    # Tuner Configuration
    tuner_init_kwargs: dict = {},

    # Analysis & Plotting
    tuner_to_df_fn: Callable = lambda tuner: tuner.get_brief_dataframe(),
    process_df_fn: Callable = lambda df: df,
    process_df_fn_kwargs: dict = {},
    process_sns_fn: Callable = default_process_sns_fn,
    plot_fn: Callable = None,
    plot_kwargs: dict = {},
    rename_mapping: dict = {},
    matplotlib_rcParams_update: dict = {},
    process_g_fn: Callable = lambda g: g,
    plot_format: str = None,

    # Control
    dos: list = ["run", "analyze"],
    yes: bool = False,
    run_choice: str = None,                # "run", "restore", "attach", "stop_attach" (CRITICAL for non-interactive)

    # Notifications
    email_notification_to_address: str = None,
) -> dict
```

## Return Value

```python
{
    "df": pd.DataFrame,  # Processed results DataFrame
    "tuner": radas.Tuner,  # Tuner object for further analysis
    "fig": matplotlib.figure.Figure,  # Figure object (when plot_fn is provided)
}
```

The `fig` object can be used to further customize or save the plot:

```python
fig = results["fig"]
fig.savefig("my_plot.png")
```

## radas.Tuner Methods

| Method                  | Description                                |
| ----------------------- | ------------------------------------------ |
| `get_results()`         | Returns ResultGrid with all trial results  |
| `get_dataframe()`       | Returns full results DataFrame             |
| `get_brief_dataframe()` | Returns DataFrame without metadata columns |

## ResultGrid Methods

```python
results_grid = tuner.get_results()
results_grid.get_best_result(metric="score", mode="max")
results_grid.get_dataframe()
```

## Result Object Properties

```python
best_result = results_grid.get_best_result()
best_result.config      # Configuration dict
best_result.metrics     # Final metrics dict
best_result.path        # Path to trial results
best_result.checkpoint  # Checkpoint if saved
```

## Ray Tune Search Space Functions

```python
from ray import tune

tune.grid_search([1, 2, 3])       # Try all values
tune.choice([1, 2, 3])            # Random choice
tune.uniform(0, 1)                # Uniform [0, 1]
tune.loguniform(1e-4, 1e-1)       # Log-uniform
tune.randint(0, 10)               # Random integer [0, 10)
tune.quniform(0, 10, 2)           # Quantized uniform
tune.sample_from(lambda: ...)     # Custom sampling
```

## Runtime Environment

The `runtime_env` parameter specifies dependencies and environment configuration for Ray workers, for example:

```python
runtime_env = {
    "pip": ["package1", "package2>=1.0"],
    "env_vars": {"KEY": "VALUE"},
    ...
}
```

Full options and details below:

### working_dir

working_dir (str): Specifies the working directory for the Ray workers. Radas automatically sets this to the current working directory where you run your script (i.e., where you `cd` to before running `python`). Do not specify this field yourself.

### py_modules

py_modules (List[str|module]): Specifies Python modules to be available for import in the Ray workers. (For more ways to specify packages, see also the pip and conda fields below.) Each entry must be either (1) a path to a local file or directory, (2) a URI to a remote zip or wheel file (see Remote URIs for details), (3) a Python module object, or (4) a path to a local .whl file.
Examples of entries in the list:
"."
"/local_dependency/my_dir_module"
"/local_dependency/my_file_module.py"
"s3://bucket/my_module.zip"
my_module # Assumes my_module has already been imported, e.g. via 'import my_module'
my_module.whl
"s3://bucket/my_module.whl"
The modules will be downloaded to each node on the cluster.

Note: Setting options (1), (3) and (4) per-task or per-actor is currently unsupported, it can only be set per-job (i.e., in ray.init()).

Note: For option (1), by default, if the local directory contains a .gitignore and/or .rayignore file, the specified files are not uploaded to the cluster. To disable the .gitignore from being considered, set RAY_RUNTIME_ENV_IGNORE_GITIGNORE=1 on the machine doing the uploading.

### py_executable

py_executable (str): Specifies the executable used for running the Ray workers. It can include arguments as well. The executable can be located in the working_dir. This runtime environment is useful to run workers in a custom debugger or profiler as well as to run workers in an environment set up by a package manager like UV (see here).
Note: py_executable is new functionality and currently experimental. If you have some requirements or run into any problems, raise issues in github.

### excludes

excludes (List[str]): Specifies a list of files or paths to exclude from being uploaded to the cluster (relative to the working_dir that radas sets internally). This field uses the pattern-matching syntax used by .gitignore files: see https://git-scm.com/docs/gitignore for details. Note: In accordance with .gitignore syntax, if there is a separator (/) at the beginning or middle (or both) of the pattern, then the pattern is interpreted relative to the level of the working_dir. In particular, you shouldn’t use absolute paths (e.g. /Users/my_working_dir/subdir/) with excludes; rather, you should use the relative path /subdir/ (written here with a leading / to match only the top-level subdir directory, rather than all directories named subdir at all levels.)
Example: {"excludes": ["my_file.txt", "/subdir/", "path/to/dir", "*.log"]}

### pip

pip (dict | List[str] | str): Either (1) a list of pip requirements specifiers, (2) a string containing the path to a local pip “requirements.txt” file, or (3) a python dictionary that has three fields: (a) packages (required, List[str]): a list of pip packages, (b) pip_check (optional, bool): whether to enable pip check at the end of pip install, defaults to False. (c) pip_version (optional, str): the version of pip; Ray will spell the package name “pip” in front of the pip_version to form the final requirement string. (d) pip_install_options (optional, List[str]): user-provided options for pip install command. Defaults to ["--disable-pip-version-check", "--no-cache-dir"]. The syntax of a requirement specifier is defined in full in PEP 508. This will be installed in the Ray workers at runtime. Packages in the preinstalled cluster environment will still be available. To use a library like Ray Serve or Ray Tune, you will need to include "ray[serve]" or "ray[tune]" here. The Ray version must match that of the cluster.
Example: ["requests==1.0.0", "aiohttp", "ray[serve]"]
Example: "./requirements.txt"
Example: {"packages":["tensorflow", "requests"], "pip_check": False, "pip_version": "==22.0.2;python_version=='3.8.11'"}
When specifying a path to a requirements.txt file, the file must be present on your local machine and it must be a valid absolute path or relative filepath relative to your local current working directory, not relative to the working_dir specified in the runtime_env. Furthermore, referencing local files within a requirements.txt file isn’t directly supported (e.g., -r ./my-laptop/more-requirements.txt, ./my-pkg.whl). Instead, use the ${RAY_RUNTIME_ENV_CREATE_WORKING_DIR} environment variable in the creation process. For example, use -r ${RAY_RUNTIME_ENV_CREATE_WORKING_DIR}/my-laptop/more-requirements.txt or ${RAY_RUNTIME_ENV_CREATE_WORKING_DIR}/my-pkg.whl to reference local files, while ensuring they’re in the working_dir.

### uv

uv (dict | List[str] | str): Alpha version feature. This plugin is the uv pip version of the pip plugin above. If you are looking for uv run support with pyproject.toml and uv.lock support, use the uv run runtime environment plugin instead.
Either (1) a list of uv requirements specifiers, (2) a string containing the path to a local uv “requirements.txt” file, or (3) a python dictionary that has three fields: (a) packages (required, List[str]): a list of uv packages, (b) uv_version (optional, str): the version of uv; Ray will spell the package name “uv” in front of the uv_version to form the final requirement string. (c) uv_check (optional, bool): whether to enable pip check at the end of uv install, default to False. (d) uv_pip_install_options (optional, List[str]): user-provided options for uv pip install command, default to ["--no-cache"]. To override the default options and install without any options, use an empty list [] as install option value. The syntax of a requirement specifier is the same as pip requirements. This will be installed in the Ray workers at runtime. Packages in the preinstalled cluster environment will still be available. To use a library like Ray Serve or Ray Tune, you will need to include "ray[serve]" or "ray[tune]" here. The Ray version must match that of the cluster.

Example: ["requests==1.0.0", "aiohttp", "ray[serve]"]
Example: "./requirements.txt"
Example: {"packages":["tensorflow", "requests"], "uv_version": "==0.4.0;python_version=='3.8.11'"}
When specifying a path to a requirements.txt file, the file must be present on your local machine and it must be a valid absolute path or relative filepath relative to your local current working directory, not relative to the working_dir specified in the runtime_env. Furthermore, referencing local files within a requirements.txt file isn’t directly supported (e.g., -r ./my-laptop/more-requirements.txt, ./my-pkg.whl). Instead, use the ${RAY_RUNTIME_ENV_CREATE_WORKING_DIR} environment variable in the creation process. For example, use -r ${RAY_RUNTIME_ENV_CREATE_WORKING_DIR}/my-laptop/more-requirements.txt or ${RAY_RUNTIME_ENV_CREATE_WORKING_DIR}/my-pkg.whl to reference local files, while ensuring they’re in the working_dir.

### conda

conda (dict | str): Either (1) a dict representing the conda environment YAML, (2) a string containing the path to a local conda “environment.yml” file, or (3) the name of a local conda environment already installed on each node in your cluster (e.g., "pytorch_p36") or its absolute path (e.g. "/home/youruser/anaconda3/envs/pytorch_p36") . In the first two cases, the Ray and Python dependencies will be automatically injected into the environment to ensure compatibility, so there is no need to manually include them. The Python and Ray version must match that of the cluster, so you likely should not specify them manually. Note that the conda and pip keys of runtime_env cannot both be specified at the same time—to use them together, please use conda and add your pip dependencies in the "pip" field in your conda environment.yaml.
Example: {"dependencies": ["pytorch", "torchvision", "pip", {"pip": ["pendulum"]}]}
Example: "./environment.yml"
Example: "pytorch_p36"
Example: "/home/youruser/anaconda3/envs/pytorch_p36"
When specifying a path to a environment.yml file, the file must be present on your local machine and it must be a valid absolute path or a relative filepath relative to your local current working directory, not relative to the working_dir specified in the runtime_env. Furthermore, referencing local files within a environment.yml file isn’t directly supported (e.g., -r ./my-laptop/more-requirements.txt, ./my-pkg.whl). Instead, use the ${RAY_RUNTIME_ENV_CREATE_WORKING_DIR} environment variable in the creation process. For example, use -r ${RAY_RUNTIME_ENV_CREATE_WORKING_DIR}/my-laptop/more-requirements.txt or ${RAY_RUNTIME_ENV_CREATE_WORKING_DIR}/my-pkg.whl to reference local files, while ensuring they’re in the working_dir.

### env_vars

env_vars (Dict[str, str]): Environment variables to set. Environment variables already set on the cluster will still be visible to the Ray workers; so there is no need to include os.environ or similar in the env_vars field. By default, these environment variables override the same name environment variables on the cluster. You can also reference existing environment variables using ${ENV_VAR} to achieve the appending behavior. If the environment variable doesn’t exist, it becomes an empty string "".
Example: {"OMP_NUM_THREADS": "32", "TF_WARNINGS": "none"}
Example: {"LD_LIBRARY_PATH": "${LD_LIBRARY_PATH}:/home/admin/my_lib"}
Non-existent variable example: {"ENV_VAR_NOT_EXIST": "${ENV_VAR_NOT_EXIST}:/home/admin/my_lib"} -> ENV_VAR_NOT_EXIST=":/home/admin/my_lib".

### nsight

nsight (Union[str, Dict[str, str]]): specifies the config for the Nsight System Profiler. The value is either (1) “default”, which refers to the default config, or (2) a dict of Nsight System Profiler options and their values. See here for more details on setup and usage.
Example: "default"
Example: {"stop-on-exit": "true", "t": "cuda,cublas,cudnn", "ftrace": ""}

### image_uri

[WIP, don't use for now] image_uri (dict): Require a given Docker image. The worker process runs in a container with this image. - Example: {"image_uri": "anyscale/ray:2.31.0-py39-cpu"}
Note: image_uri is experimental. If you have some requirements or run into any problems, raise issues in github.
**Important**: The `image_uri` field currently cannot be used together with other fields of runtime_env (e.g., `working_dir`, `py_modules`, `excludes`, `pip`, etc.). If you need to use `image_uri`, it must be the only field in runtime_env.

## Integrations

### Weights & Biases (wandb)

```python
# wandb_trainable.py
def trainable(config):
    from radas.rwandb import rwandb_init, rwandb_log, rwandb_finish

    run = rwandb_init(config)
    score = config["a"] + config["b"]
    rwandb_log({"score": score})
    rwandb_finish()
    return {"score": score}
```

```python
results = await run_experiment(
    trainable=trainable,
    param_space={
        "wandb": {
            "enable": True,
            "api_key": "your_wandb_api_key",
            "init_kwargs": {"entity": "your-entity"},
        },
        "a": tune.grid_search([1, 2]),
    },
)
```

### SwanLab

Similar to wandb, use `radas.rswanlab` module.

## Data Storage

Two ways to use storage on the cluster:

### Option 1: Use Path (`radas.get_data_dir`)

Get a shared data directory path and use standard Python file I/O operations.

```python
import radas
import os

data_dir = radas.get_data_dir(user_name="your_username")

# Standard file I/O
file_path = os.path.join(data_dir, "test_file.txt")
with open(file_path, "w") as f:
    f.write("This is a test file.")
with open(file_path, "r") as f:
    content = f.read()
```

- Returns a shared data directory path accessible across cluster nodes
- Use standard Python file operations (open, read, write)
- Good for working with raw files directly

Also available: `radas.get_cold_data_dir(user_name="your_username")` for cold storage.

### Option 2: Use Artifact (`radas.cloud_artifact`)

Higher-level API for serializing/deserializing Python objects.

```python
import radas.cloud_artifact as rda

rda.init(user_name="your_username")

# Save Python objects
rda.save_cloud_artifact("my_dict.pkl", {"a": 1, "b": 2})

# Load Python objects
loaded_dict = rda.load_cloud_artifact("my_dict.pkl")
```

- Handles pickling automatically (based on .pkl extension)
- Good for saving/loading Python objects like dicts, models, arrays, etc.
- Storage is accessible across cluster nodes

## Schedulers

```python
from ray.tune.schedulers import (
    ASHAScheduler,
    HyperBandScheduler,
    PopulationBasedTraining,
    MedianStoppingRule,
)

scheduler = ASHAScheduler(
    metric="score",
    mode="max",
    grace_period=10,
    max_t=100,
    reduction_factor=2,
)
```

## Search Algorithms

```python
from ray.tune.search.optuna import OptunaSearch
from ray.tune.search.hyperopt import HyperOptSearch
from ray.tune.search.bayesopt import BayesOptSearch

search_alg = OptunaSearch(
    metric="score",
    mode="max",
)
```

## Cloud Artifact API

```python
import radas.cloud_artifact as rda

rda.init(user_name="your_username")
rda.save_cloud_artifact("path/to/file.pkl", data)
data = rda.load_cloud_artifact("path/to/file.pkl")
rda.rm_cloud_artifact("path/to/file.pkl")
```

## Email API

```python
from radas.email import send_email

send_email(
    to_address="recipient@example.com",
    subject="Subject line",
    body="Email body text",
)
```

## Cluster API

### Check Cluster Availability

```python
from radas.clusters import get_cluster_availability

# Get status and recommendations for all clusters
print(get_cluster_availability())
# Output:
# All available clusters are listed below, with status and suggestions of when to use which cluster:
# - [status: online] atol-gpu-5090: 32G GRAM per GPU, choose as the default choice, best performance, maximum availibility
# - [status: offline] atol-gpu-pro6000: 96G GRAM, only use when you need GRAM > 32G per GPU, availability is limited
```

### Cluster Resource Specifications

```python
from radas.clusters.addresses import gpu_per_cpu, cpu_per_node, gpu_per_node

# Access cluster resource specifications
gpu_per_cpu["atol-gpu-5090"]   # GPUs allocated per CPU
cpu_per_node["atol-gpu-5090"]  # Total CPUs per node
gpu_per_node["atol-gpu-5090"]  # Total GPUs per node
```

## Utility Functions

```python
from radas.core import get_ray_results_dir

# Get the path where Ray stores experiment results on the cluster
# Returns e.g. "/mnt/data/nfs/ray-results/yuhang/"
results_dir = get_ray_results_dir()
```

```python
from radas.utils import (
    set_seed,              # Set random seed for reproducibility
    config_seed,           # Set seed from config dict
    get_current_user_name, # Get system username
    print_info,            # Print info message
    print_warning,         # Print warning message
    print_error,           # Print error message
)

from radas import (
    explode,               # Explode list columns in DataFrame
    explode_tensor,        # Convert tensor to DataFrame
    explode_dict_of_tensors,  # Convert dict of tensors to DataFrame
)
```

## DataFrame Processing Functions

```python
# Reduce by config columns
from radas.utils import reduce_by_config_cols, reduce_by_reduced_col

# Reduce scores by config columns
df_reduced = reduce_by_config_cols(
    df=df,
    metric_col="score",
    reduce_fn=lambda series: series.mean(),
    config_cols=["config/a"],
)
```
