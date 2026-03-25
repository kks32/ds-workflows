# Parameter Sweep with PyLauncher

This workflow runs a parameter sweep using dapi's parametric sweep interface. PyLauncher dispatches many independent serial tasks across all cores in a single SLURM allocation, eliminating the overhead of submitting separate jobs for each parameter combination. The workflow covers defining parameter grids, previewing combinations, generating launcher files, submitting, and collecting results.

## Initialize the Client

```python
from dapi import DSClient

ds = DSClient()
```

## Define the Parameter Sweep

A sweep is a dictionary where each key is a parameter name and each value is a list of values to test. dapi generates the full Cartesian product of all parameters.

```python
sweep = {
    "ALPHA": [0.3, 0.5, 3.7],
    "BETA":  [1.1, 2.0, 3.0],
}
```

This 3x3 grid produces 9 parameter combinations. Each combination becomes an independent serial task.

## Preview the Combinations

Before writing any files to disk, preview the generated commands to verify the parameter substitution looks correct.

```python
ds.jobs.parametric_sweep.generate(
    'python3 simulate.py --alpha ALPHA --beta BETA',
    sweep,
    preview=True,
)
```

This prints every command that would appear in `runsList.txt` without creating files. You should see 9 lines, each with different ALPHA and BETA values substituted into the template. Fix any issues with the command template before proceeding.

## Generate the Sweep Files

Once the preview looks correct, generate the actual files. The third argument is the local path where the files will be written.

```python
ds.jobs.parametric_sweep.generate(
    'python3 simulate.py --alpha ALPHA --beta BETA '
    '--output "$WORK/sweep_$SLURM_JOB_ID/run_ALPHA_BETA"',
    sweep,
    "/home/jupyter/MyData/sweep_demo/",
    debug="host+job",
)
```

Two files appear in `/home/jupyter/MyData/sweep_demo/`.

- `runsList.txt` contains one command per parameter combination, with placeholders replaced by actual values.
- `call_pylauncher.py` is the wrapper script that PyLauncher reads at runtime.

The `$WORK` and `$SLURM_JOB_ID` variables are not evaluated at generation time. They expand on the compute node when the job runs, so each task writes output to a unique directory on the fast work filesystem.

The `debug="host+job"` option adds diagnostic output to each task so you can confirm which node and job ID each task ran under.

## Submit the Sweep

```python
job = ds.jobs.parametric_sweep.submit(
    "/MyData/sweep_demo/",
    app_id="designsafe-agnostic-app",
    allocation="your_allocation",
    node_count=1,
    cores_per_node=48,
    max_minutes=30,
)
job.monitor()
```

With 48 cores and 9 tasks, all tasks run concurrently on the first pass. If you had 200 tasks, PyLauncher would cycle through them as cores become available, assigning the next task to each core as it finishes.

Match `cores_per_node` to the target machine. Frontera CLX nodes have 56 cores, Lonestar6 nodes have 128, and Stampede3 nodes have 48.

## Check Results

```python
job.print_runtime_summary()

outputs = job.list_outputs()
for f in outputs:
    print(f.name)
```

Read the main job log to verify PyLauncher completed without errors.

```python
stdout = job.get_output_content("tapisjob.out")
print(stdout)
```

## Understanding PyLauncher Output

PyLauncher creates per-task output within the job directory. Each task's stdout and stderr are captured separately. A few things to keep in mind.

- Per-task log files often have no file extension. Data Depot may not display them correctly. Use JupyterHub to browse and read these files instead.
- Task output directories follow the pattern you specified in the command template. In the example above, each task writes to `$WORK/sweep_$SLURM_JOB_ID/run_<ALPHA>_<BETA>`.
- If a task fails, it does not stop other tasks. PyLauncher continues dispatching the remaining tasks. Check individual task logs to identify failures.

## OpenSees Cantilever Pushover Sweep

A more realistic use case follows. This sweep varies nodal mass (5 values) and column length (3 values) for a 2D cantilever pushover analysis, producing 15 independent runs.

### Define the Parameters

```python
sweep = {
    "NODAL_MASS": [4.19, 4.39, 4.59, 4.79, 4.99],
    "LCOL":       [100, 200, 300],
}
```

### Preview and Generate

```python
# Preview first
ds.jobs.parametric_sweep.generate(
    'python3 cantilever_pushover.py '
    '--mass NODAL_MASS --lcol LCOL '
    '--outDir "$WORK/sweep_$SLURM_JOB_ID/run_NODAL_MASS_LCOL"',
    sweep,
    preview=True,
)
```

This should show 15 commands. Once confirmed, generate the files.

```python
ds.jobs.parametric_sweep.generate(
    'python3 cantilever_pushover.py '
    '--mass NODAL_MASS --lcol LCOL '
    '--outDir "$WORK/sweep_$SLURM_JOB_ID/run_NODAL_MASS_LCOL"',
    sweep,
    "/home/jupyter/MyData/opensees_sweep/",
    debug="host+job",
)
```

### Submit and Monitor

```python
job = ds.jobs.parametric_sweep.submit(
    "/MyData/opensees_sweep/",
    app_id="designsafe-agnostic-app",
    allocation="your_allocation",
    node_count=1,
    cores_per_node=48,
    max_minutes=60,
)
job.monitor()
```

With 15 tasks and 48 cores, all 15 runs fit on a single node. Each run writes to its own output directory on the work filesystem.

### Output Files

Each run produces the following files in its output directory.

| File | Contents |
|---|---|
| `DFree.out` | Displacement at the free node (top of the cantilever) over time |
| `RBase.out` | Base reaction forces over time |
| `FCol.out` | Column element forces at each time step |

These files are space-delimited with one row per time step. Read them with `numpy.loadtxt()` or `pandas.read_csv()` using whitespace as the separator.

### Collecting Results Across Runs

After the sweep finishes, you can loop over parameter combinations and read results from each output directory.

```python
import numpy as np

results = []
for mass in sweep["NODAL_MASS"]:
    for lcol in sweep["LCOL"]:
        path = f"run_{mass}_{lcol}/DFree.out"
        content = job.get_output_content(path)
        data = np.loadtxt(content.split("\n"))
        max_disp = np.max(np.abs(data[:, 1]))
        results.append({
            "mass": mass,
            "lcol": lcol,
            "max_displacement": max_disp,
        })

import pandas as pd
df = pd.DataFrame(results)
print(df)
```

## Complete Workflow (Simple Sweep)

```python
from dapi import DSClient

ds = DSClient()

sweep = {
    "ALPHA": [0.3, 0.5, 3.7],
    "BETA":  [1.1, 2.0, 3.0],
}

# Preview
ds.jobs.parametric_sweep.generate(
    'python3 simulate.py --alpha ALPHA --beta BETA '
    '--output "$WORK/sweep_$SLURM_JOB_ID/run_ALPHA_BETA"',
    sweep,
    preview=True,
)

# Generate files
ds.jobs.parametric_sweep.generate(
    'python3 simulate.py --alpha ALPHA --beta BETA '
    '--output "$WORK/sweep_$SLURM_JOB_ID/run_ALPHA_BETA"',
    sweep,
    "/home/jupyter/MyData/sweep_demo/",
    debug="host+job",
)

# Submit and monitor
job = ds.jobs.parametric_sweep.submit(
    "/MyData/sweep_demo/",
    app_id="designsafe-agnostic-app",
    allocation="your_allocation",
    node_count=1,
    cores_per_node=48,
    max_minutes=30,
)
job.monitor()

# Results
job.print_runtime_summary()
job.list_outputs()
print(job.get_output_content("tapisjob.out"))
```

## Related

- [dapi documentation](https://designsafe-ci.github.io/dapi/) for the full parametric sweep API reference
- [SLURM PyLauncher](../hpc/pylauncher.md) for how PyLauncher works at the SLURM level
