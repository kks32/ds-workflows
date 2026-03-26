# Parameter Sweeps

A structural model needs to be tested with 5 material strengths and 10 load levels. That produces 50 independent runs. Submitting each one by hand through the [DesignSafe](https://designsafe-ci.org) portal would take all afternoon. A parameter sweep automates this by generating all 50 runs and executing them concurrently on the allocated compute nodes.

Parameter sweeps are common in sensitivity analysis, uncertainty quantification, and Monte Carlo studies where each run is independent and writes its own output. A bridge fragility study might vary ground-motion intensity across 500 records. A geotechnical analysis might sweep friction angles from 20 to 40 degrees in 1-degree increments. The pattern is always the same. One model, many parameter combinations, no communication between runs.

## How PyLauncher Works

[PyLauncher](https://github.com/TACC/pylauncher) is a lightweight task launcher developed at [TACC](https://www.tacc.utexas.edu/) for running many independent serial tasks inside a single [SLURM](https://slurm.schedmd.com/documentation.html) allocation. Instead of submitting hundreds of separate jobs (each waiting in the queue independently), one job is submitted and PyLauncher dispatches tasks across the allocated cores until the full list is exhausted.

This approach avoids repeated queue waits, reduces scheduler load, and keeps the allocated nodes busy. A sweep of 500 tasks on a 48-core node runs roughly 48 tasks at a time, with PyLauncher feeding new tasks to cores as previous ones finish.

## When to Use PyLauncher

PyLauncher fits when there are many independent serial runs, each run writes to its own output directory, and there is no communication between tasks. Typical examples include running the same OpenSeesPy model with different material parameters, sweeping load magnitudes, or varying geometric configurations across dozens of cases.

## When Not to Use PyLauncher

If individual runs need [MPI](https://www.mpi-forum.org/) (tightly coupled parallel execution), PyLauncher's `ClassicLauncher` is not the right tool. Use a native MPI job configuration instead. Also avoid PyLauncher when runs share output files, since multiple tasks writing to the same path will cause race conditions or data loss.

## The dapi Workflow

The [dapi](https://designsafe-ci.github.io/dapi/) library provides a `parametric_sweep` interface that generates the PyLauncher input files and submits the job.

First, preview the sweep to verify parameters before generating files.

```python
ds.jobs.parametric_sweep.generate(
    app_id="openseespy-s3",
    input_dir_uri=input_uri,
    script_filename="model.py",
    sweep_params={"NodalMass": [11.79, 11.99, 12.19]},
    node_count=1,
    cores_per_node=48,
    max_minutes=60,
    allocation="your_allocation",
    queue="skx",
    preview=True,
)
```

The preview displays the commands that PyLauncher will run without creating any files. Once the parameters look right, remove the `preview=True` flag, then submit.

```python
sweep = ds.jobs.parametric_sweep.generate(
    app_id="openseespy-s3",
    input_dir_uri=input_uri,
    script_filename="model.py",
    sweep_params={"NodalMass": [11.79, 11.99, 12.19]},
    node_count=1,
    cores_per_node=48,
    max_minutes=60,
    allocation="your_allocation",
    queue="skx",
)

job = ds.jobs.parametric_sweep.submit(sweep)
job.monitor()
```

PyLauncher reads a command list (one line per task) and dispatches each command to an available core. Each task runs in its own temporary directory, and results are copied to the specified output directories.

## Comparison of Approaches

Three strategies exist for running many independent tasks on TACC systems. The right choice depends on task count, task duration, and how much control is needed.

| Feature | PyLauncher | Separate [Tapis](https://tapis.readthedocs.io/en/latest/) Jobs | SLURM Job Arrays |
|---|---|---|---|
| Queue waits | One wait for the entire batch | One wait per job | Elements scheduled independently |
| Scheduler load | Low (one SLURM job) | High (one SLURM job per run) | Moderate (many sub-jobs under one array ID) |
| Task granularity | One command per line, arbitrarily complex | Each job is fully independent | Each element uses `$SLURM_ARRAY_TASK_ID` |
| Fault tolerance | A node failure kills the entire allocation. Restart files are available but require setup. | Each job is independent. Resubmit individual failures. | Each element is independent. Resubmit individual indices. |
| Ideal task count | Hundreds to hundreds of thousands | Fewer than ~50 | Tens to thousands |
| Ideal task duration | Seconds to minutes | Minutes to hours | Minutes to hours |
| TACC recommendation | Preferred for high-throughput parameter sweeps | Acceptable for small batches | Supported on [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/), but arrays at large scale increase scheduler pressure |

For short tasks numbering in the thousands (each finishing in under a minute), PyLauncher generally provides the best time-to-solution because it amortizes scheduler overhead across all tasks. For longer tasks (15 minutes or more each), the performance difference between PyLauncher and job arrays becomes negligible.

## Common Mistakes

All runs writing to the same directory. Each task must write to a unique output folder. If two tasks write `results.txt` to the same path, one overwrites the other. Always include an output directory argument in each command (for example, `--outDir outCase41`).

Expecting output in `tapisjob.out`. PyLauncher captures stdout and stderr per task, not globally. Application output will not appear in the main Tapis output file. Instead, look inside the PyLauncher-created subdirectory in the input directory. Each task has its own log files. Use JupyterHub to inspect these files, since Data Depot may not render files without extensions correctly.

Forgetting to set `UseMPI=false`. When submitting a PyLauncher job through a Tapis app, set `UseMPI = false`. Enabling MPI for serial tasks causes jobs to hang, crash, or behave inconsistently.

Using PyLauncher for tightly coupled workflows. PyLauncher dispatches independent tasks. If tasks need to communicate with each other during execution, use MPI directly instead. [Debugging HPC Jobs](debugging.md) covers MPI in detail.

## Further Reading

- [dapi parametric_sweep API reference](https://designsafe-ci.github.io/dapi/)
- [PyLauncher at TACC](https://docs.tacc.utexas.edu/software/pylauncher/) for SLURM-level configuration details and advanced launcher modes
