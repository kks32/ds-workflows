# HPC Job Scheduling with SLURM

Every job submitted through [dapi](https://designsafe-ci.github.io/dapi/) or the DesignSafe web portal ultimately runs on [TACC](https://www.tacc.utexas.edu/) supercomputers managed by [SLURM](https://slurm.schedmd.com/documentation.html) (Simple Linux Utility for Resource Management). SLURM is the open-source job scheduler used on Stampede3, Frontera, Lonestar6, and most major supercomputing centers worldwide.

When you call `ds.jobs.submit()` in dapi, [Tapis](https://tapis.readthedocs.io/en/latest/) translates your job request into a SLURM batch script, stages your input files to the execution system, and runs `sbatch` on your behalf. Understanding what SLURM does with that script helps you choose resources wisely, debug failures, and scale your workflows.

## What SLURM does

SLURM manages five things for every job.

1. It schedules jobs across the supercomputer cluster based on requested resources and queue priority.
2. It allocates hardware (CPU cores, memory, GPUs) from a pool shared by all users.
3. It queues jobs when the requested resources are not immediately available.
4. It monitors execution and enforces time limits.
5. It logs results, exit codes, and resource usage for accounting and reproducibility.

## How a job flows through the system

When you submit a job from DesignSafe (whether through dapi, the web portal, or direct SSH), it passes through several layers before computation begins.

1. Your job request specifies an application, input files, and resource requirements (nodes, cores, walltime, allocation).
2. Tapis packages this into a SLURM batch script and stages your input files to the execution system (typically Stampede3).
3. Tapis runs `sbatch` on a login node of the execution system. The job enters the SLURM queue and receives a priority based on the fair-share scheduling policy.
4. When sufficient resources become available, SLURM allocates compute nodes and launches your batch script.
5. Your application runs on the allocated compute nodes. Output files are written to the job's working directory.
6. After completion, Tapis archives the output files back to your DesignSafe storage.

You can monitor this entire lifecycle from dapi with `job.monitor()`, which polls the Tapis job status until the job reaches a terminal state (FINISHED, FAILED, or CANCELLED).

## DesignSafe apps vs. direct SLURM access

Most users never write SLURM scripts directly. DesignSafe apps and dapi generate the correct SLURM configuration automatically based on the application and your resource requests.

Direct SLURM access (via SSH to a TACC login node) becomes useful when you need fine-grained control over MPI launch configuration, custom module loading, multi-step job scripts that chain preprocessing and simulation, or advanced scheduler features like job arrays and dependencies.

The rest of this section explains SLURM concepts that apply regardless of how the job was submitted.

## SLURM commands reference

If you work directly on a TACC login node via SSH, these are the primary SLURM commands.

| Command | Purpose |
|---|---|
| `sbatch job.slurm` | Submit a batch job |
| `squeue -u $USER` | List your queued and running jobs |
| `squeue -j JOBID` | Check a specific job |
| `scancel JOBID` | Cancel a job |
| `sacct -j JOBID` | View accounting details after completion |
| `sinfo` | Display available partitions and node status |

When using dapi, these commands run automatically through Tapis. The `job.get_status()` and `job.monitor()` methods in dapi correspond to `squeue` and `sacct` queries.
