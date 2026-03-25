# Job Resources

When you submit a job through [dapi](https://designsafe-ci.github.io/dapi/) or the DesignSafe web portal, you specify resource parameters that determine how your job runs on TACC systems. These parameters map directly to [SLURM](https://slurm.schedmd.com/documentation.html) batch directives. Choosing them well reduces queue wait time and prevents premature termination.

## Parameters you control

| Parameter | dapi argument | What it controls |
|---|---|---|
| Allocation | `allocation` | The TACC project account charged for the job |
| Queue | `queue` | Which partition the job runs in (affects max nodes, time limits, priority) |
| Max runtime | `max_minutes` | Wall-clock time limit in minutes. Jobs exceeding this are killed. |
| Node count | `node_count` | Number of physical machines allocated |
| Cores per node | `cores_per_node` | CPU cores per node. Total cores = node_count x cores_per_node. |
| Memory | `memory_mb` | Memory in megabytes (optional, system defaults usually suffice) |

## How dapi maps to SLURM

When you call `ds.jobs.generate()`, dapi builds a [Tapis](https://tapis.readthedocs.io/en/latest/) job request. Tapis then generates a SLURM batch script with `#SBATCH` directives matching your parameters.

```python
job_request = ds.jobs.generate(
    app_id="opensees-mp-s3",
    input_dir_uri=input_uri,
    script_filename="model.tcl",
    node_count=2,
    cores_per_node=48,
    max_minutes=120,
    allocation="your_allocation",
    queue="normal",
)
```

This produces a SLURM script equivalent to

```bash
#!/bin/bash
#SBATCH -A your_allocation
#SBATCH -p normal
#SBATCH -N 2
#SBATCH -n 96
#SBATCH -t 02:00:00
# ... module loads, execution commands ...
```

You never write this script yourself. Tapis generates it from the app definition and your overrides.

## Choosing the right resources

**Walltime.** Start with a generous estimate and tighten after your first successful run. File staging (input transfer to the compute system, output archival afterward) counts toward walltime. If your simulation takes 30 minutes but staging takes 10, request at least 50 minutes.

**Nodes and cores.** For MPI jobs (OpenSees MP, ADCIRC), total MPI ranks = node_count x cores_per_node. Match this to your domain decomposition. For serial jobs or [PyLauncher](pylauncher.md) sweeps, one node is usually sufficient.

**Queue.** The `development` queue has shorter time limits but less wait. Use it for testing. The `normal` queue handles production runs. Each queue has maximum node counts and walltime limits that vary by system. See the [TACC user guides](https://docs.tacc.utexas.edu/) for current limits.

**Memory.** Most jobs run fine with system defaults. Override `memory_mb` only if your application fails with out-of-memory errors.

## TACC system quick reference

| System | Cores per node | Memory per node | Common queues |
|---|---|---|---|
| [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/) (SPR) | 112 | 128 GB | normal, development, large |
| [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/) (SKX) | 48 | 192 GB | skx-normal, skx-dev |
| [Frontera](https://docs.tacc.utexas.edu/hpc/frontera/) | 56 | 192 GB | normal, development, large |
| [Lonestar6](https://docs.tacc.utexas.edu/hpc/lonestar6/) | 128 | 256 GB | normal, development |

## What happens to your files

Tapis handles file staging automatically. Your input directory is copied to a working directory on the execution system's shared filesystem before the job starts. After completion, output files are archived back to your DesignSafe storage. Both transfers count toward walltime.

If your job produces many small files, the archival step can be slow. Bundling outputs into a tar or zip file inside your script reduces this overhead. See [Job Profiling](../tapis/job-profiling.md) for more on file transfer performance.

## Direct SLURM access

If you SSH to a TACC login node and submit jobs with `sbatch` directly, you write the full SLURM script yourself. The [TACC documentation](https://docs.tacc.utexas.edu/) covers this workflow in detail. The resource parameters are identical to those described above, expressed as `#SBATCH` directives.

Common commands for direct SLURM use

| Command | Purpose |
|---|---|
| `sbatch job.slurm` | Submit a batch job |
| `squeue -u $USER` | List your queued and running jobs |
| `scancel JOBID` | Cancel a job |
| `sacct -j JOBID` | View accounting details after completion |
| `idev -N 1 -n 48 -p development -t 00:30:00` | Start an interactive development session |
